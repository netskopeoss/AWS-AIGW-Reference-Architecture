# CLAUDE_DEV.md

Development instructions for Claude Code when modifying code or templates in this repository.

## Template

This repository contains a single production template: `templates/gateway-combined.yaml`.
It deploys AI Gateway (AIG) + DLP On Demand (DLPoD) + optional AI Guardrails in one stack.

> **Standalone templates** (AIG-only, DLPoD-only) are in [AWS-POV-Templates-CFT](https://github.com/jharris-ns/AWS-POV-Templates-CFT).

## Directory Structure

```
templates/
  gateway-combined.yaml       # Combined AIG + DLPoD + optional Guardrails

docs/
  DEPLOYMENT.md               # Deployment guide
  OPERATIONS.md               # Operations reference
  ARCHITECTURE.md             # Architecture narrative
  QUICKSTART.md               # Condensed quick-start reference
  SECURITY.md                 # Secret handling, IAM, network security
  TROUBLESHOOTING.md          # Failure diagnosis and recovery

scripts/
  deploy-artifacts.sh         # Creates S3 bucket for template upload
```

There is no `dist/` directory and no Lambda packaging step — all Lambdas are inline.

## Lifecycle Flows

### AIG

The AIG lifecycle uses an inline Activation Lambda — no Step Functions, no SSH:

1. ASG launches instance → lifecycle hook holds it in `Pending:Wait`
2. SNS delivers event to `AigActivationLambdaFunction` (inline)
3. Lambda registers appliance with Netskope API → receives enrollment token
4. Lambda writes bootstrap secret with `enrollment_token` plus the DLPoD cert and host
   (and `ai_guardrails.host` when Guardrails is deployed)
5. Lambda calls `CompleteLifecycleAction(CONTINUE)` → instance moves to `InService`
6. Instance reads bootstrap secret on first boot → self-enrolls using aig-cli
7. On termination: Lambda deregisters appliance, deletes SSM parameter

### DLPoD

DLPoD uses nsbootstrap.service with EC2 UserData — no SSH, no Step Functions, no paramiko:

1. `CertGeneratorFunction` (inline Lambda) generates self-signed TLS certs at stack create
2. `DlpodBootstrapBuilderFunction` (inline Lambda) reads certs + license key from Secrets Manager,
   assembles `bootstrap.json`, and returns base64-encoded UserData
3. ASG launches DLPoD instance with UserData containing the encoded `bootstrap.json`
4. `nsbootstrap.service` reads `bootstrap.json` at first boot and configures TLS certs,
   license key, DNS, and persona — instance is ready without any external orchestration

## Key Resources

Full inventory is in `templates/gateway-combined.yaml`; the resources most often touched:

| Resource | Type | Purpose |
|----------|------|---------|
| `GatewayAutoScalingGroup` | AutoScaling::AutoScalingGroup | AIG instance management |
| `GatewayLaunchTemplate` | EC2::LaunchTemplate | AIG instance config (AMI, SG, IAM, EBS) |
| `GatewayAlb` | ELBv2::LoadBalancer | Internet-facing HTTPS ingress |
| `AigActivationLambdaFunction` | Lambda::Function | AIG registration + bootstrap (inline) |
| `AigBootstrapSecret` | SecretsManager::Secret | Enrollment token + DLP cert (written at launch) |
| `NetskopeSecret` | SecretsManager::Secret | Tenant URL + API token |
| `AigAlbCertificate` | Custom::AlbCertificate | Self-signed cert for AIG ALB (if no ACM cert) |
| `DlpodAutoScalingGroup` | AutoScaling::AutoScalingGroup | DLPoD instance management |
| `DlpodBootstrapBuilderFunction` | Lambda::Function | Assembles bootstrap.json UserData (inline) |
| `CertGeneratorFunction` | Lambda::Function | Generates self-signed TLS certs (inline, shared) |
| `DlpodAlbCertificate` | Custom::AlbCertificate | Self-signed cert for DLPoD ALB |
| `DlpodCredentialsSecret` | SecretsManager::Secret | DLPoD license key |
| `DlpodPrivateHostedZone` | Route53::HostedZone | Private zone `aigw.internal` (DLPoD + Guardrails records) |
| `DlpodReadinessGateFunction` | Lambda::Function | Target-group readiness poller (inline; DLPoD gate only, 840 s budget) |
| `DlpodReadinessGate` | Custom::DlpodReadiness | Blocks AIG launch until DLPoD targets healthy |
| `GuardrailsAutoScalingGroup` | AutoScaling::AutoScalingGroup | *(Condition: DeployGuardrails)* GPU instances running `aisecurityllm`; `HealthCheckType: EC2` |
| `GuardrailsLaunchTemplate` | EC2::LaunchTemplate | *(conditional)* DL Base GPU AMI; UserData mounts local NVMe, does `aws s3 cp` + `docker load` + `docker run` with Docker data-root on NVMe, then polls `/ping` |
| `GuardrailsAlb` | ELBv2::LoadBalancer | *(conditional)* Internal HTTP ALB at `guardrails.aigw.internal` |
| `GuardrailsWaitHandle` | CloudFormation::WaitConditionHandle | *(conditional)* Pre-signed URL passed into Guardrails UserData |
| `GuardrailsReadinessGate` | CloudFormation::WaitCondition | *(conditional)* Blocks AIG launch until the first Guardrails instance signals SUCCESS from UserData (`Timeout: 3600`, `Count: 1`; UserData signals FAILURE after 15 min) |

## Template Conventions

- **YAML only**, two-space indent
- All named resources use `!Sub '${AWS::StackName}-<role>'`
- No tag parameters — `Project`, `Environment`, and `ManagedBy` are passed as stack-level `--tags`
- IAM follows least-privilege — separate statements per permission grant, no `Resource: '*'`
  except where the API requires it: `ec2:DescribeInstances` (Activation Lambda),
  `acm:ImportCertificate` / `DeleteCertificate` / `AddTagsToCertificate` (cert generator),
  and `elasticloadbalancing:DescribeTargetHealth` (readiness gate)
- Sensitive values in Secrets Manager. `GatewayRole` may read only `AigBootstrapSecret`
  (`ReadBootstrapSecret`); no instance role can read `NetskopeSecret` (the API credentials)
- Lifecycle hooks must be **inline** on the ASG (`LifecycleHookSpecificationList`) — separate
  resources create a race condition where instances launch before hooks exist
- All Lambda functions use inline `ZipFile` code — no S3 Lambda artifacts are required.
  This keeps templates self-contained without a packaging step.

## Artifacts

All Lambda functions are inline (`ZipFile`) — no packaged artifacts to build or upload.
The only script is `scripts/deploy-artifacts.sh`, which creates the S3 bucket used for
template upload (required because the template, ~71 KB, exceeds 51 KB). Override the bucket
name with `TEMPLATE_BUCKET=<name>`.

## Development Rules

- **Lifecycle hooks must be inline** on the ASG — separate `AWS::AutoScaling::LifecycleHook`
  resources create a race condition (instances launch before hooks exist).
- **ASG must DependsOn SNS subscription and Lambda permission** — prevents instances from
  launching before the lifecycle event delivery chain is wired.
- **AIG uses inline Lambda only** — the Activation Lambda is inline (`ZipFile`). The enrollment flow is bootstrap-secret-based (instance
  self-enrolls on boot). Do not introduce Step Functions or SSH into the AIG enrollment path
  unless reverting to the legacy pattern.
- **DLPoD uses nsbootstrap.service** — instances self-configure at first boot using
  `bootstrap.json` delivered via EC2 UserData. No SSH, no paramiko, no Step Functions.
- **Keep CloudFormation conventions** — explicit IAM policies with no `Action: '*'`; no tag
  parameters (pass `Project`/`Environment`/`ManagedBy` via `--tags` at deploy time).
- **Do not hardcode AMI IDs or IP addresses** in documentation — environment-specific, passed
  as parameters.
- **Do not store API credentials on instances** — Activation Lambda handles all Netskope API
  calls. Enrollment token is passed via bootstrap secret and never persisted elsewhere.
- **Cert must have `CA:TRUE` basicConstraints** for the AIG DLP service to accept it.
- **Lambdas must never log secret values** — there is no masking layer; the functions simply
  do not print `api_token`, `enrollment_token`, `license_key`, or TLS private keys. Keep it
  that way when adding log lines.
- **Template size**: the template (~71 KB) exceeds 51 KB → must be deployed via
  `--template-url` referencing S3.
- **No `UpdatePolicy` on any ASG** — an AMI or launch-template change does not replace
  instances; operators run `aws autoscaling start-instance-refresh` manually. Do not document
  automatic instance refresh unless an `UpdatePolicy` is added.
- **Guardrails instance types must have local NVMe instance storage** — UserData places the
  image tarball and Docker data-root on the instance store (falling back to `/tmp` on the gp3
  root, which is much slower and risks the 15-minute UserData budget). Do not add EBS-only
  types to `GuardrailsInstanceType` AllowedValues, and do not remove the NVMe mount /
  Docker data-root logic from Guardrails UserData.
- **Guardrails is optional and conditional** — every Guardrails resource carries
  `Condition: DeployGuardrails`. `DependsOn` cannot target a conditional resource (cfn-lint
  E3005), so `GatewayAutoScalingGroup` depends on `GuardrailsReadinessGate` via a
  `!If [DeployGuardrails, !Ref GuardrailsReadinessGate, disabled]` tag value instead.
- **Custom::AlbCertificate must `!Ref` its SSM placeholder** (`CertParameterName: !Ref
  DlpodCertParameter`), not `!Sub` the name. The Lambda writes the parameter with
  `Overwrite=True`; without the implicit dependency CloudFormation can try to create the
  placeholder afterwards and fail with `ParameterAlreadyExists`.
- **Run `cfn-lint templates/gateway-combined.yaml` before committing** — the template is
  currently lint-clean (W2506/W1030 suppressed for the String-typed optional `GuardrailsAmiId`).
