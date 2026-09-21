# Operations Guide — AI Gateway + DLP On Demand

Day-2 operations reference for `templates/gateway-combined.yaml`. Written for DevOps engineers
who know AWS well but may be new to Netskope. Background on what the appliances are and how the
stack is laid out lives in [ARCHITECTURE.md](ARCHITECTURE.md); this guide is about operating a
running stack.

Shell examples in this document assume:

```bash
STACK=<stack-name>
REGION=<region>
```

## Table of Contents

- [Background](#background)
- [Startup Sequence](#startup-sequence)
- [Monitoring and Alerts](#monitoring-and-alerts)
- [Key Operational Commands](#key-operational-commands)
- [Scaling](#scaling)
- [AMI Upgrade Procedure](#ami-upgrade-procedure)
- [Certificate Renewal](#certificate-renewal)
- [IAM Roles](#iam-roles)
- [Secrets and SSM Parameters](#secrets-and-ssm-parameters)
- [Troubleshooting](#troubleshooting)

---

## Background

### What Is the AI Gateway?

The Netskope AI Gateway (AIG) is a software appliance that runs as an EC2 instance in your VPC and
acts as an inline, OpenAI-compatible HTTPS proxy between AI applications and LLM providers. It
enrolls with the Netskope management plane at boot using a one-time token that the Activation
Lambda places in Secrets Manager, forwards content to DLP On Demand for inspection, receives
policies from the tenant, and records every AI interaction there. See
[ARCHITECTURE.md — AI Gateway tier](ARCHITECTURE.md#ai-gateway-aig--internet-facing-tier).

### What Is DLP On Demand?

DLP On Demand (DLPoD) is a content-inspection appliance that the AIG forwards content to over
HTTPS:443 at `dlp.aigw.internal`. It runs in your VPC — content never leaves your AWS account for
DLP scanning. It self-configures at first boot from a `bootstrap.json` document delivered in EC2
UserData (TLS cert + key, license key, DNS resolver, `dlp-on-demand` persona); there is no SSH,
lifecycle hook, or orchestration service. Once licensed it connects outbound to the Netskope
management plane via the NAT Gateway and downloads DLP profiles. See
[ARCHITECTURE.md — DLP On Demand tier](ARCHITECTURE.md#dlp-on-demand-dlpod--internal-inspection-tier).

### Layout

One VPC, two AZs. AIG instances (ASG `<stack>-aig-asg`, min 1 / max 4) sit behind an
internet-facing ALB in the public subnets; DLPoD instances (ASG `<stack>-dlpod-asg`, min 1 / max 4)
sit behind an internal ALB in the private subnets, reached via the Route 53 private zone
`aigw.internal`. All instances are in private subnets and egress through one NAT Gateway. The
optional AI Guardrails tier (ASG `<stack>-guardrails-asg`, GPU instances behind an internal HTTP
ALB at `guardrails.aigw.internal:8080`) is present only when `GuardrailsImageS3Bucket` was set.
Diagrams and per-tier detail: [ARCHITECTURE.md](ARCHITECTURE.md#architecture-diagram).

---

## Startup Sequence

Stack creation is strictly ordered: DLPoD (and Guardrails, if deployed) must be serving before the
first AIG instance launches. `GatewayAutoScalingGroup` has `DependsOn: DlpodReadinessGate` (and
`GuardrailsReadinessGate` when deployed). The AIG validates the DLPoD HTTPS endpoint and the
Guardrails host during enrollment, so these gates guarantee the endpoints exist before AIG boots.
Total creation time is 12–18 minutes (DLPoD bootstrap ~5–10 min, then AIG enrollment ~5–15 min);
longer with Guardrails.

For the full traffic-flow context of each sequence, see
[ARCHITECTURE.md — Traffic Flows](ARCHITECTURE.md#traffic-flows). The same ordering is described
from the deployment perspective in [DEPLOYMENT.md — Startup Ordering](DEPLOYMENT.md#startup-ordering).

### Pre-launch: certificate and bootstrap UserData (stack creation only)

Before any instances launch, two custom resources establish the shared configuration that both
services depend on:

```
Stack creation
  ├─ 1. DlpodAlbCertificate (Custom::AlbCertificate → CertGeneratorFunction, <stack>-certgen)
  │       a. Generates a two-tier cert hierarchy: CA + leaf (CN=dlp.aigw.internal, 365-day validity)
  │       b. Imports the leaf cert + key to ACM → CertificateArn used by the DLPoD ALB listener
  │       c. Writes the CA cert PEM to SSM Parameter Store: /<stack>/dlpod-cert
  │       d. Writes CA cert + leaf cert + leaf key to Secrets Manager: <stack>-dlpod-cert-key
  │
  └─ 2. DlpodBootstrapPart1 / DlpodBootstrapPart2 (Custom::DlpodBootstrap → DlpodBootstrapBuilderFunction)
          a. Reads leaf cert + key + CA cert from <stack>-dlpod-cert-key
          b. Reads the license key from <stack>-dlpod-credentials
          c. Assembles bootstrap.json:
               { "dlpaas": { "server-cert", "server-key", "server-intermediate-ca-chain" },
                 "dns": { "primary": "169.254.169.253" },
                 "system": { "licensekey": "<key>" },
                 "persona": "dlp-on-demand" }
          d. Base64-encodes it and returns it in two halves (custom resource responses are
             limited to 4 KB); DlpodLaunchTemplate re-joins them as the instance UserData
```

The AIG side does not read the bootstrap UserData. Instead, the Activation Lambda reads the
CA cert from `/<stack>/dlpod-cert` at every AIG instance launch and writes it, together with the
fixed DLP host `https://dlp.aigw.internal`, into the AIG bootstrap secret (see the AIG flow below).

> If the certificate step fails the stack rolls back — nothing else can be created without it. If
> the bootstrap builder fails, DLPoD instances never receive a valid `bootstrap.json` and never
> become healthy. See [TROUBLESHOOTING.md — Certificate Issues](TROUBLESHOOTING.md#certificate-issues)
> and [TROUBLESHOOTING.md — DLP On Demand Issues](TROUBLESHOOTING.md#dlp-on-demand-issues).

---

### DLPoD bootstrap flow (per instance, ~5–10 min to ALB-healthy)

Runs for every DLPoD instance that launches — at stack creation and on every scale-out or
replacement. There is no lifecycle hook, Lambda, or orchestration per instance: the launch
template UserData already contains everything the appliance needs.

```
Instance launch
  │
  ├─ 1. ASG launches DLPoD instance from DlpodLaunchTemplate
  │       └─ Instance enters InService immediately (no lifecycle hook)
  │       └─ HealthCheckType: ELB, HealthCheckGracePeriod: 1800s / 30 min
  │           If the ALB health check is still failing after 30 min → ASG marks the instance
  │           Unhealthy and launches a replacement
  │
  ├─ 2. nsbootstrap.service reads bootstrap.json from EC2 UserData at first boot
  │       └─ Installs the TLS server cert, key, and CA chain (dlpaas block)
  │       └─ Sets the DNS resolver to 169.254.169.253 (Route 53 Resolver — resolves AWS
  │           endpoints and the aigw.internal private zone)
  │       └─ Applies the license key (system.licensekey)
  │       └─ Sets the persona to dlp-on-demand
  │       └─ Appliance connects outbound to the Netskope management plane via the NAT Gateway
  │
  └─ 3. DLPoD service starts listening on HTTPS:443
           └─ DLPoD ALB health check (HTTPS GET / on 443, any 200–499 response,
               2 consecutive passes at 30 s) → target healthy
           └─ Instance begins receiving DLP inspection traffic from AIG
```

The ALB health check confirms only that the appliance answers on 443. Netskope's setup guide
states the appliance needs roughly 30 minutes after tethering to fully initialise (download DLP
profiles and report ready), so a target can be healthy in the ALB before inspection is fully
effective. Confirm appliance status in the Netskope console (Security Cloud Platform >
On-Premises Infrastructure) after a new instance appears healthy.

**DLPoD readiness gate (stack creation only):** after `DlpodAutoScalingGroup` is created, the
`DlpodReadinessGate` custom resource (inline Lambda `<stack>-dlpod-readiness`, log group
`/aws/lambda/<stack>-dlpod-readiness`) polls `describe-target-health` on the DLPoD target group
every 30 seconds. It returns `SUCCESS` when all targets are healthy and `FAILED` (rolling the stack
back) if that has not happened within 840 seconds (14 minutes; the Lambda's own timeout is 900 s).
The failure reason in `describe-stack-events` reads
`Targets in <stack>-dlpod-tg did not become healthy within 840s`. `GatewayAutoScalingGroup`
depends on this resource, so no AIG instance launches until DLPoD is serving. The gate only runs
on stack create — it is a no-op on update and delete, and does not apply to later scale-outs.

**On termination:** There is no termination hook. The instance is terminated by the ASG and the
ALB deregisters the target. The Netskope management plane detects the appliance disconnect.

---

### Guardrails startup flow (per instance, only when deployed)

Each Guardrails instance runs the launch-template UserData at boot, logging to
`/var/log/user-data.log`:

```
Instance launch
  │
  ├─ 1. ASG launches GPU instance from GuardrailsLaunchTemplate (HealthCheckType: EC2, no grace period)
  ├─ 2. UserData: nvidia-smi (fail fast if the driver is missing)
  ├─ 3. UserData mounts the local NVMe instance store (/opt/dlami/nvme) — falls back to /tmp on
  │      the EBS root if no instance store is found (much slower; see the note below)
  ├─ 4. UserData installs Docker / NVIDIA Container Toolkit if absent, moves Docker data-root onto
  │      the NVMe mount, downloads s3://<GuardrailsImageS3Bucket>/<GuardrailsImageS3Key> to it,
  │      docker load, then docker run --gpus all --restart=unless-stopped -p 8080:8080
  ├─ 5. UserData polls http://localhost:8080/ping every 10 s, up to 90 times (15 min)
  │      └─ 200 → curl PUT SUCCESS to the GuardrailsWaitHandle URL
  │      └─ timeout or any script error → curl PUT FAILURE
  └─ 6. Guardrails ALB health check (HTTP GET /ping, 200, 30 s interval, 2 passes) → target healthy
```

**Guardrails readiness gate (stack creation only):** `GuardrailsReadinessGate` is an
`AWS::CloudFormation::WaitCondition` (`Count: 1`, `Timeout: 3600`). It is satisfied by the first
Guardrails instance's SUCCESS signal and fails the stack (rollback) if a FAILURE signal arrives or
no signal arrives within 60 minutes. In practice the UserData's own 15-minute limit is what fires.
This gate does not use the readiness Lambda, so nothing about it appears in
`/aws/lambda/<stack>-dlpod-readiness`; the signal `Reason` (for example
`Container not healthy after 15 min`) is recorded in `describe-stack-events`, and the detail is in
`/var/log/user-data.log` on the instance (SSM Session Manager). Like the DLPoD gate, it only
matters during stack creation — later launches still run the same UserData and still `curl` the
handle URL, but CloudFormation ignores signals to a completed WaitCondition.

---

### AIG enrollment flow (per instance, ~5–15 min)

Triggered for every AIG instance that launches — at stack creation and on every scale-out or
replacement. The lifecycle hook heartbeat is only 120 seconds, so the Activation Lambda must
complete quickly.

```
Instance launch
  │
  ├─ 1. ASG launches AIG instance
  │       └─ Lifecycle hook holds instance in Pending:Wait (HeartbeatTimeout: 120s / 2 min)
  │           If the Activation Lambda does not complete within 2 min → instance ABANDONED
  │
  ├─ 2. ASG lifecycle event → SNS topic → Activation Lambda (AigActivationLambdaFunction,
  │       function name <stack>-aig-activation)
  │       │
  │       ├─ a. Reads API credentials from Secrets Manager (<stack>-netskope-credentials)
  │       │         { "tenant_url": "...", "api_token": "..." }
  │       │
  │       ├─ b. Calls Netskope REST API: POST /api/v2/aig/appliances
  │       │         Registers the appliance (name <stack>-gw-<instance-id>, host = private IP) → receives:
  │       │           - id  (Netskope's identifier for this appliance)
  │       │           - enrollment_token  (one-time token; used by the instance at boot)
  │       │
  │       ├─ c. Writes appliance id to AWS Systems Manager Parameter Store
  │       │         Path: /aig/<stack>/<instance-id>
  │       │         Purpose: the termination Lambda invocation is a fresh execution with no memory
  │       │         of the launch — it needs the id to call DELETE /api/v2/aig/appliances/{id}.
  │       │         SSM is the bridge between the two invocations.
  │       │
  │       ├─ d. Reads the DLPoD CA cert PEM from SSM: /<stack>/dlpod-cert
  │       │         (written by CertGeneratorFunction at stack creation)
  │       │
  │       ├─ e. Writes the bootstrap secret (<stack>-aig-bootstrap) — full overwrite:
  │       │           { "bootstrap": true,
  │       │             "enrollment_token": "<token>",
  │       │             "dlp": { "host": "https://dlp.aigw.internal", "certificate": "<CA PEM>" },
  │       │             "ai_guardrails": { "host": "http://guardrails.aigw.internal:8080/invocations" } }
  │       │         The ai_guardrails block is present only when GuardrailsImageS3Bucket was set.
  │       │
  │       └─ f. Calls CompleteLifecycleAction: CONTINUE
  │                 Instance moves from Pending:Wait → InService.
  │                 Any exception → CompleteLifecycleAction: ABANDON (instance is replaced).
  │
  └─ 3. AIG instance boots and reads bootstrap secret from Secrets Manager
           └─ UserData: {"bootstrap_secret": "<stack>-aig-bootstrap"}
           └─ Self-enrolls with Netskope tenant using enrollment_token
           └─ Configures DLP forwarding to https://dlp.aigw.internal using the DLP block
           └─ Starts the AI Gateway service
           └─ ALB health check passes (HTTPS GET / on port 443, 10 s interval) → serving requests
               (HealthCheckType: ELB, HealthCheckGracePeriod: 600s)
```

**On termination:** The AIG termination lifecycle hook fires → the Activation Lambda reads the
appliance id from `/aig/<stack>/<instance-id>` → calls `DELETE /api/v2/aig/appliances/{id}` to
deregister from the Netskope tenant → deletes the SSM parameter → completes the lifecycle hook
(`CONTINUE` even if deregistration fails, so termination is never blocked).

> **Shared bootstrap secret:** every launch overwrites `<stack>-aig-bootstrap` with that instance's
> enrollment token. Scale AIG one instance at a time — two instances launching concurrently can
> read each other's token. The DLP (and Guardrails) block is identical for all instances, so it is
> unaffected by this.

> **DLP traffic:** AIG instances configure DLP forwarding from their first boot. At stack creation
> the readiness gate guarantees DLPoD is healthy first. On later AIG scale-outs, if the DLPoD ALB
> has no healthy targets, the AIG continues serving requests but DLP inspection is not applied
> until a healthy DLPoD target is available.

---

## Monitoring and Alerts

### Log Groups

The stack creates four CloudWatch log groups, one per inline Lambda function. This table is the
canonical reference; SECURITY.md links here.

| Log group | Contents | Retention | Typical volume |
|---|---|---|---|
| `/aws/lambda/<stack>-aig-activation` | AIG enrollment/deregistration events, Netskope API calls, lifecycle hook completion | 30 days | ~10 lines per instance launch/termination |
| `/aws/lambda/<stack>-certgen` | Cert hierarchy generation, ACM import, SSM and Secrets Manager writes | 30 days | ~5 lines at stack creation (and delete) only |
| `/aws/lambda/<stack>-dlpod-bootstrap-builder` | `bootstrap.json` assembly — size of the base64 UserData and each half, never contents | 30 days | ~2 lines at stack creation/update only |
| `/aws/lambda/<stack>-dlpod-readiness` | DLPoD readiness gate polls: `N/M target(s) healthy — waiting 30s...` | 7 days | ~1 line per 30 s during stack creation only |

The Guardrails readiness gate is a WaitCondition signalled from instance UserData and writes to
none of these groups; its output is `/var/log/user-data.log` on the Guardrails instance.

DLPoD instances themselves write no CloudWatch logs from the stack's perspective —
`nsbootstrap.service` runs on the appliance. Its status is observable only through the DLPoD ALB
target health (below) or on the appliance itself; consult the Netskope DLP On Demand
documentation for appliance-side diagnostics.

Tail any log group in real time:
```bash
aws logs tail /aws/lambda/$STACK-aig-activation --follow --region $REGION
```

### Built-in CloudWatch Alarm

The stack creates one CloudWatch alarm automatically:

| Alarm | Metric | Threshold | Action |
|---|---|---|---|
| `<stack>-aig-high-cpu` | `AWS/EC2 CPUUtilization`, Average, on the AIG ASG | Strictly greater than `ScaleOutCpuThreshold` (default 70 %) for 2 consecutive 5-min periods (`GreaterThanThreshold`) | Step scaling — adds one AIG instance |

Check alarm state:
```bash
aws cloudwatch describe-alarms --alarm-names $STACK-aig-high-cpu \
  --query "MetricAlarms[0].[StateValue,StateReason]" --output table --region $REGION
```

The alarm sets `TreatMissingData: notBreaching`, so its steady state is `OK` even at minimum
capacity with little traffic (missing datapoints are treated as within threshold rather than
producing `INSUFFICIENT_DATA`). `ALARM` triggers scale-out. There is no scale-in alarm or policy —
scale-in is manual (see [Scaling](#scaling)).

### Recommended Additional Alarms

These alarms are not created by the stack but are useful for production deployments:

```bash
# Alert when AIG ASG has fewer instances than desired (instance failures)
aws cloudwatch put-metric-alarm \
  --alarm-name "$STACK-aig-below-desired" \
  --metric-name GroupInServiceInstances \
  --namespace AWS/AutoScaling \
  --dimensions Name=AutoScalingGroupName,Value=$STACK-aig-asg \
  --statistic Minimum --period 300 --threshold 1 \
  --comparison-operator LessThanThreshold --evaluation-periods 1 \
  --alarm-description "AIG ASG in-service count below desired" \
  --region $REGION

# Alert when DLPoD ALB has no healthy targets
aws cloudwatch put-metric-alarm \
  --alarm-name "$STACK-dlpod-no-healthy-targets" \
  --metric-name HealthyHostCount \
  --namespace AWS/ApplicationELB \
  --dimensions \
    Name=LoadBalancer,Value=<dlpod-alb-arn-suffix> \
    Name=TargetGroup,Value=<dlpod-tg-arn-suffix> \
  --statistic Minimum --period 300 --threshold 1 \
  --comparison-operator LessThanThreshold --evaluation-periods 1 \
  --alarm-description "DLPoD has no healthy targets — DLP inspection unavailable" \
  --region $REGION

# Alert on Activation Lambda errors
aws cloudwatch put-metric-alarm \
  --alarm-name "$STACK-aig-activation-errors" \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --dimensions Name=FunctionName,Value=$STACK-aig-activation \
  --statistic Sum --period 300 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold --evaluation-periods 1 \
  --alarm-description "Activation Lambda errors — enrollment failures" \
  --region $REGION

# (Guardrails only) Alert when a Guardrails target is unhealthy — the ASG will NOT replace it
aws cloudwatch put-metric-alarm \
  --alarm-name "$STACK-guardrails-unhealthy-target" \
  --metric-name UnHealthyHostCount \
  --namespace AWS/ApplicationELB \
  --dimensions \
    Name=LoadBalancer,Value=<guardrails-alb-arn-suffix> \
    Name=TargetGroup,Value=<guardrails-tg-arn-suffix> \
  --statistic Maximum --period 300 --threshold 0 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 2 \
  --alarm-description "Guardrails target unhealthy — manual replacement required" \
  --region $REGION
```

The `<...-arn-suffix>` values are the `app/<name>/<id>` and `targetgroup/<name>/<id>` portions of
the ALB and target group ARNs (`aws elbv2 describe-load-balancers` / `describe-target-groups`).

---

## Key Operational Commands

Resolve the target group ARNs once and reuse them in the commands below:

```bash
AIG_TG=$(aws elbv2 describe-target-groups --names $STACK-aig-tg \
  --query "TargetGroups[0].TargetGroupArn" --output text --region $REGION)
DLPOD_TG=$(aws elbv2 describe-target-groups --names $STACK-dlpod-tg \
  --query "TargetGroups[0].TargetGroupArn" --output text --region $REGION)
# Guardrails only
GUARDRAILS_TG=$(aws elbv2 describe-target-groups --names $STACK-guardrails-tg \
  --query "TargetGroups[0].TargetGroupArn" --output text --region $REGION)
```

### AI Gateway

AIG ASG instance states:
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-aig-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION
```

AIG ALB target health:
```bash
aws elbv2 describe-target-health --target-group-arn $AIG_TG --output table --region $REGION
```

Activation Lambda logs (look for `Registered appliance`, `Traceback`, `ABANDON`):
```bash
aws logs tail /aws/lambda/$STACK-aig-activation --since 30m --region $REGION
```

AIG bootstrap secret (contains the most recent enrollment token — treat as sensitive):
```bash
aws secretsmanager get-secret-value --secret-id $STACK-aig-bootstrap \
  --query SecretString --output text --region $REGION
```

Enrolled appliances: Netskope console, **Settings → Security Cloud Platform → AI Gateway**.

### DLP On Demand

DLPoD ASG instance states:
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-dlpod-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION
```

DLPoD ALB target health:
```bash
aws elbv2 describe-target-health --target-group-arn $DLPOD_TG --output table --region $REGION
```

DLPoD bootstrap builder logs (stack create/update only):
```bash
aws logs tail /aws/lambda/$STACK-dlpod-bootstrap-builder --since 1h --region $REGION
```

DLPoD readiness gate logs (stack create only):
```bash
aws logs tail /aws/lambda/$STACK-dlpod-readiness --since 1h --region $REGION
```

Readiness gate result from stack events (both gates):
```bash
aws cloudformation describe-stack-events --stack-name $STACK \
  --query "StackEvents[?LogicalResourceId=='DlpodReadinessGate' || LogicalResourceId=='GuardrailsReadinessGate'].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]" \
  --output table --region $REGION
```

DLPoD CA cert (public PEM, from SSM):
```bash
aws ssm get-parameter --name /$STACK/dlpod-cert --query Parameter.Value \
  --output text --region $REGION
```

DLPoD service URL: `https://dlp.aigw.internal` (fixed; `DlpodServiceUrl` stack output) —
resolvable only inside the VPC.

### AI Guardrails (only when `GuardrailsImageS3Bucket` was set)

Guardrails ASG instance states:
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-guardrails-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION
```

Guardrails ALB target health:
```bash
aws elbv2 describe-target-health --target-group-arn $GUARDRAILS_TG --output table --region $REGION
```

Shell on a Guardrails instance (no SSH key; the role has `AmazonSSMManagedInstanceCore`):
```bash
aws ssm start-session --target <instance-id> --region $REGION
```

On the instance — container status, logs, UserData log, and local health check:
```bash
sudo docker ps
sudo docker logs guardrails
cat /var/log/user-data.log
curl -s http://localhost:8080/ping        # expect "Healthy"
```

Confirm the NVMe fast path was used (if the Docker root dir is under `/tmp`, the instance store
was not found and the slow EBS fallback is in effect):
```bash
lsblk -dpno NAME,MODEL
mountpoint /opt/dlami/nvme
sudo docker info | grep 'Docker Root Dir'
```

Health check from an AIG instance's point of view (run from any host in the VPC):
```bash
curl -s http://guardrails.aigw.internal:8080/ping
```

Scale Guardrails:
```bash
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name $STACK-guardrails-asg \
  --desired-capacity <N> --region $REGION
```

**Health checks and replacement.** The Guardrails ASG uses `HealthCheckType: EC2` with no grace
period. The ALB health check (`/ping`, HTTP 200) controls only whether a target receives traffic;
it does not feed back into the ASG. Consequently, an instance whose container has crashed or hung
stays `InService`/`Healthy` in the ASG and is never replaced automatically — only an EC2
status-check failure triggers replacement. Detect this with `describe-target-health` (target
`unhealthy` while the ASG shows `Healthy`) or the `UnHealthyHostCount` alarm above, then replace
the instance yourself:

```bash
# Option A — replace every instance in the ASG one at a time (also picks up launch-template changes)
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name $STACK-guardrails-asg --region $REGION

# Option B — replace one instance; the ASG launches a substitute
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id <instance-id> --no-should-decrement-desired-capacity --region $REGION
```

Guardrails instances have no lifecycle hook: a replacement pulls the image and starts the container
from UserData, then the ALB health check governs when it receives traffic.

**Boot time and the NVMe instance store.** UserData budgets 15 minutes for the container to pass
`/ping`, and that budget assumes the image tarball and Docker `data-root` are on the local NVMe
instance store (which is why `GuardrailsInstanceType` is limited to `g4dn`/`g5` sizes). The
instance store is ephemeral, so every launch — scale-out, replacement, or refresh — re-downloads
`aisecurity-llm.tgz` from S3 and re-loads it. Keep the bucket in the stack's region; if boots are
slow, run the `lsblk` / `docker info` check above to confirm the fast path is in use.

Roll to a new container image: upload the new tarball to S3 under a new key, `update-stack` with
the new `GuardrailsImageS3Key` (all other parameters `UsePreviousValue=true` — see the
[AMI Upgrade Procedure](#ami-upgrade-procedure) for the full parameter list), then
`start-instance-refresh` on `<stack>-guardrails-asg`. A stack update alone does not replace
instances.

---

## Scaling

All three ASGs are fixed at `MinSize: 1` / `MaxSize: 4`; the only automatic policy is AIG
scale-out on CPU. Everything else below is a manual `set-desired-capacity`.

### Scale AIG out manually

```bash
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name $STACK-aig-asg \
  --desired-capacity <new-count> \
  --region $REGION
```

Each new AIG instance goes through the full enrollment flow (~5–15 minutes). The Activation
Lambda writes the enrollment token to the bootstrap secret and completes the lifecycle hook —
enrollment happens autonomously on the instance from there. Increase desired capacity by one at
a time: all AIG instances share the single `<stack>-aig-bootstrap` secret, and concurrent launches
can pick up each other's enrollment token.

### Scale DLPoD out manually

```bash
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name $STACK-dlpod-asg \
  --desired-capacity <new-count> \
  --region $REGION
```

Each new DLPoD instance bootstraps independently from the same launch template UserData
(~5–10 minutes until the ALB health check passes; longer until fully initialised in the Netskope
console). DLPoD instances do not conflict — the `bootstrap.json` is identical for every instance
and contains no per-instance state. The readiness gate does not run on scale-out; watch the DLPoD
ALB target health to know when the new instance is serving. Concurrent DLPoD launches are safe.

### Scale in

```bash
# AIG scale in
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name $STACK-aig-asg \
  --desired-capacity <new-count> --region $REGION

# DLPoD scale in
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name $STACK-dlpod-asg \
  --desired-capacity <new-count> --region $REGION
```

There is no scale-in policy in the template, so capacity added by the CPU alarm stays until you
reduce it. On AIG scale-in the termination lifecycle hook fires — the Activation Lambda deregisters
the appliance from the Netskope tenant and deletes the SSM appliance ID parameter. DLPoD has no
termination hook — the instance is simply terminated and deregistered from the ALB.

### Update desired capacity via stack update

The desired counts are parameters and can be changed persistently with a stack update (a manual
`set-desired-capacity` is reverted on the next stack update that touches the ASG). Every parameter
without a default must be carried forward with `UsePreviousValue=true`; the Guardrails parameters
default to empty, so omitting them on a stack that has Guardrails deployed tears the tier down:

```bash
aws cloudformation update-stack \
  --stack-name $STACK \
  --template-url https://<bucket>.s3.$REGION.amazonaws.com/templates/gateway-combined.yaml \
  --parameters \
    ParameterKey=NetskopeTenantUrl,UsePreviousValue=true \
    ParameterKey=NetskopeApiToken,UsePreviousValue=true \
    ParameterKey=DlpodLicenseKey,UsePreviousValue=true \
    ParameterKey=AcmCertificateArn,UsePreviousValue=true \
    ParameterKey=GatewayAmiId,UsePreviousValue=true \
    ParameterKey=DlpodAmiId,UsePreviousValue=true \
    ParameterKey=GuardrailsImageS3Bucket,UsePreviousValue=true \
    ParameterKey=GuardrailsImageS3Key,UsePreviousValue=true \
    ParameterKey=GuardrailsAmiId,UsePreviousValue=true \
    ParameterKey=DesiredCapacity,ParameterValue=<new-aig-count> \
    ParameterKey=DlpodDesiredCapacity,ParameterValue=<new-dlpod-count> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION
```

---

## AMI Upgrade Procedure

When Netskope releases a new AIG or DLPoD AMI version, upgrade in two steps: update the stack
parameter (which creates a new launch-template version), then start an instance refresh on the
ASG. **No ASG in the template has an `UpdatePolicy`, so a stack update by itself does not replace
running instances** — they keep running the old AMI until you refresh.

**1. Find the new AMI ID:**
```bash
aws ec2 describe-images \
  --filters 'Name=name,Values=*Netskope AI Gateway*' \
  --query 'sort_by(Images, &CreationDate)[-1].[ImageId,Name,CreationDate]' \
  --output table --region $REGION
```

For DLPoD the AMI is shared privately to your account from the Netskope console, so list private
images instead (`aws ec2 describe-images --executable-users self ...`) or read the ID from
EC2 > AMIs > Private images.

**2. Update the stack with the new AMI:**

Carry every existing parameter forward with `UsePreviousValue=true` — `NetskopeTenantUrl` has no
default (the update fails without it), and the three Guardrails parameters default to empty
(omitting them on a stack with Guardrails deployed deletes the tier). Change only the AMI you are
upgrading:

```bash
aws cloudformation update-stack \
  --stack-name $STACK \
  --template-url https://<bucket>.s3.$REGION.amazonaws.com/templates/gateway-combined.yaml \
  --parameters \
    ParameterKey=NetskopeTenantUrl,UsePreviousValue=true \
    ParameterKey=NetskopeApiToken,UsePreviousValue=true \
    ParameterKey=DlpodLicenseKey,UsePreviousValue=true \
    ParameterKey=AcmCertificateArn,UsePreviousValue=true \
    ParameterKey=DesiredCapacity,UsePreviousValue=true \
    ParameterKey=DlpodDesiredCapacity,UsePreviousValue=true \
    ParameterKey=DlpodAmiId,UsePreviousValue=true \
    ParameterKey=GuardrailsImageS3Bucket,UsePreviousValue=true \
    ParameterKey=GuardrailsImageS3Key,UsePreviousValue=true \
    ParameterKey=GuardrailsAmiId,UsePreviousValue=true \
    ParameterKey=GatewayAmiId,ParameterValue=<new-aig-ami-id> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION

aws cloudformation wait stack-update-complete --stack-name $STACK --region $REGION
```

For a DLPoD upgrade swap the last line for `ParameterKey=DlpodAmiId,ParameterValue=<new-dlpod-ami-id>`
and set `GatewayAmiId,UsePreviousValue=true`. For a Guardrails AMI upgrade do the same with
`GuardrailsAmiId`.

**3. Start the instance refresh:**
```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name $STACK-aig-asg --region $REGION
# or, for the tier you upgraded:
#   --auto-scaling-group-name $STACK-dlpod-asg
#   --auto-scaling-group-name $STACK-guardrails-asg
```

The default refresh keeps 90 % of capacity healthy and replaces instances one at a time. Each AIG
replacement goes through the full enrollment flow (~5–15 min); each DLPoD replacement re-runs
`nsbootstrap` from the (unchanged) UserData (~5–10 min to ALB-healthy); each Guardrails
replacement re-downloads and loads the image (up to 15 min). Because AIG instances share one
bootstrap secret, do not lower the refresh's `MinHealthyPercentage` in a way that launches two AIG
instances at once. Plan for reduced capacity during the rollout.

**4. Monitor the refresh:**
```bash
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name $STACK-aig-asg \
  --query "InstanceRefreshes[0].[Status,PercentageComplete,StatusReason]" \
  --output table --region $REGION
```

Then confirm with `describe-target-health` on the relevant target group and, for AIG, look for
`Registered appliance` in `/aws/lambda/<stack>-aig-activation`.

---

## Certificate Renewal

The DLPoD CA and leaf certificates (and the auto-generated AIG ALB certificate, if
`AcmCertificateArn` was left empty) are valid for 365 days from stack creation. After expiry the
AIG can no longer verify the DLPoD ALB and DLP inspection fails.

A stack update that re-runs the `DlpodAlbCertificate` custom resource (for example by changing one
of its properties) regenerates the hierarchy, rewrites `/<stack>/dlpod-cert` and
`<stack>-dlpod-cert-key`, and rebuilds the DLPoD launch-template UserData. Running instances do
not pick this up on their own: start an instance refresh on `<stack>-dlpod-asg` (so the appliances
serve the new leaf) and then on `<stack>-aig-asg` (so the Activation Lambda writes the new CA into
the bootstrap secret at each relaunch). Schedule this before the anniversary of stack creation.
Verify the current expiry with:

```bash
aws ssm get-parameter --name /$STACK/dlpod-cert --query Parameter.Value \
  --output text --region $REGION | openssl x509 -noout -enddate
```

---

## IAM Roles

The eight IAM roles (seven without Guardrails), their principals, and every permission statement
are documented once in [SECURITY.md — IAM Roles and Permissions](SECURITY.md#iam-roles-and-permissions).
No role in the stack can SSH to or otherwise log in to an AIG or DLPoD instance.

---

## Secrets and SSM Parameters

The four Secrets Manager secrets and three SSM Parameter Store paths — contents, writer, reader,
and lifecycle — are documented once in
[SECURITY.md — What's Stored and Where](SECURITY.md#whats-stored-and-where). Operationally: the
only secret you normally read is `<stack>-aig-bootstrap` (to confirm the `dlp` block is present),
and the only parameter is `/<stack>/dlpod-cert` (to check certificate expiry — see
[Certificate Renewal](#certificate-renewal)).

---

## Troubleshooting

For full Issue/Cause/Solution troubleshooting, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

Quick reference for the most common issues:

| Symptom | Where to look |
|---|---|
| AIG instance stuck in `Pending:Wait` more than 2 minutes | `/aws/lambda/<stack>-aig-activation` logs |
| AIG instance ABANDONED | Activation Lambda logs; fix root cause before next replacement launches |
| DLPoD target never becomes healthy | DLPoD ALB target health; `/aws/lambda/<stack>-dlpod-bootstrap-builder` logs; DLPoD SG/ALB SG rules |
| Stack rolled back at `DlpodReadinessGate` (`Targets in <stack>-dlpod-tg did not become healthy within 840s`) | `describe-stack-events`; `/aws/lambda/<stack>-dlpod-readiness` logs; re-create with `--disable-rollback` to inspect |
| Stack rolled back at `GuardrailsReadinessGate` (`Container not healthy after 15 min` / `UserData script failed`) | `describe-stack-events`; `/var/log/user-data.log` on the Guardrails instance via SSM; check S3 bucket region and that the NVMe fast path was used — see [TROUBLESHOOTING.md](TROUBLESHOOTING.md#ai-guardrails-issues) |
| Guardrails target unhealthy but ASG instance shows `Healthy` | Expected with `HealthCheckType: EC2` — replace manually (see [AI Guardrails](#ai-guardrails-only-when-guardrailsimages3bucket-was-set)) |
| DLP inspection not working (AIG enrolled but no DLP) | Check DLPoD ALB target health; check bootstrap secret has `dlp` block; confirm appliance is fully initialised in the Netskope console |
| Scale-out alarm not triggering | `aws cloudwatch describe-alarms --alarm-names <stack>-aig-high-cpu` — steady state is `OK`, not `INSUFFICIENT_DATA` |
| New AMI parameter applied but instances still on the old AMI | Expected — no `UpdatePolicy`; run `start-instance-refresh` (see [AMI Upgrade Procedure](#ami-upgrade-procedure)) |
| Stack deletion hanging | Check for instances in `Terminating:Wait`; may need manual `CompleteLifecycleAction` |
