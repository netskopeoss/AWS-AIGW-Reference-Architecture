# Security Reference — AI Gateway + DLP On Demand

Security posture reference for `templates/gateway-combined.yaml`, which deploys Netskope
AI Gateway (AIG) and DLP On Demand (DLPoD) with an optional AI Guardrails tier. Intended for
InfoSec reviewers, security architects, and compliance teams evaluating this deployment.

This document is the canonical home for the IAM role inventory and the secrets / SSM parameter
inventory; ARCHITECTURE.md and OPERATIONS.md link here rather than repeating them.

## Table of Contents

- [Security Design Principles](#security-design-principles)
- [Network Security](#network-security)
- [IAM Roles and Permissions](#iam-roles-and-permissions)
- [Secret and Credential Management](#secret-and-credential-management)
- [Encryption](#encryption)
- [CloudFormation Security Practices](#cloudformation-security-practices)
- [Audit and Visibility](#audit-and-visibility)
- [Known Limitations and Accepted Risks](#known-limitations-and-accepted-risks)

---

## Security Design Principles

1. **Least privilege IAM** — Seven dedicated IAM roles (eight when the optional Guardrails tier is
   deployed); each has exactly the permissions its function requires. Secrets Manager, SSM, and
   Auto Scaling actions are scoped to specific resource ARNs.

2. **Secrets never touch instances directly** — The Netskope API token flows from Secrets Manager
   to the Activation Lambda only (in memory). The Lambda exchanges the token for a short-lived
   enrollment token, which it writes to the bootstrap secret. AIG instances read the bootstrap
   secret at boot — they never have access to the raw API token. DLPoD instances read no secret
   at all: their configuration is assembled once at stack creation by an inline Lambda and
   delivered as launch template UserData.

3. **No public IP on compute** — All EC2 instances run in private subnets. Inbound access is
   exclusively through the ALBs. There is no SSH inbound path to any instance — from the internet
   or from inside the VPC. No security group in the stack opens port 22.

4. **All sensitive parameters are `NoEcho`** — `NetskopeApiToken` and `DlpodLicenseKey` are
   never shown in CloudFormation events, stack outputs, or the console after submission.

5. **No orchestration or remote-login automation** — DLPoD configures itself from `bootstrap.json`
   via its own `nsbootstrap.service` at first boot. There is no orchestration service, no
   remote-login automation, and no lifecycle hook on the DLPoD Auto Scaling Group.

---

## Network Security

### Security Group Rules

**AIG ALB Security Group** (`<stack>-aig-alb-sg`)

| Direction | Protocol | Port | Source / Destination | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 443 | `0.0.0.0/0` | HTTPS from internet clients |
| Outbound | TCP | 443 | VPC CIDR | Forward to AIG instances |

**AIG Instance Security Group** (`<stack>-aig-gw-sg`)

| Direction | Protocol | Port | Source / Destination | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 443 | AIG ALB SG | HTTPS from ALB only |
| Outbound | All | All | `0.0.0.0/0` | Outbound via NAT GW (Netskope API, LLM providers, DLPoD ALB, Guardrails ALB) |

**DLPoD ALB Security Group** (`<stack>-dlpod-alb-sg`)

| Direction | Protocol | Port | Source / Destination | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 443 | AIG Instance SG | HTTPS from AIG instances only (`AigToDlpodAlbIngress`) |
| Outbound | TCP | 443 | VPC CIDR | Forward to DLPoD instances |

**DLPoD Instance Security Group** (`<stack>-dlpod-sg`)

| Direction | Protocol | Port | Source / Destination | Purpose |
|---|---|---|---|---|
| Inbound | TCP | 443 | DLPoD ALB SG | HTTPS for DLP inspection (only ingress rule — no SSH) |
| Outbound | All | All | `0.0.0.0/0` | Netskope management plane connection via NAT GW |

There is no Lambda security group: none of the stack's four Lambda functions is VPC-attached.
They call AWS APIs (ACM, SSM, Secrets Manager, ELBv2, Auto Scaling, EC2) and the Netskope REST
API over the public Lambda network path only.

**Guardrails ALB Security Group** (`<stack>-guardrails-alb-sg`) — *only when Guardrails is deployed*

| Direction | Protocol | Port | Source / Destination | Purpose |
|---|---|---|---|---|
| Inbound | TCP | `GuardrailsContainerPort` (8080) | AIG Instance SG | HTTP inference requests from AIG instances only |
| Outbound | TCP | `GuardrailsContainerPort` (8080) | VPC CIDR | Forward to Guardrails instances |

**Guardrails Instance Security Group** (`<stack>-guardrails-sg`) — *only when Guardrails is deployed*

| Direction | Protocol | Port | Source / Destination | Purpose |
|---|---|---|---|---|
| Inbound | TCP | `GuardrailsContainerPort` (8080) | Guardrails ALB SG | HTTP from ALB only |
| Outbound | All | All | `0.0.0.0/0` | S3 image download via NAT GW |

### Network Isolation

- **No direct internet inbound to instances.** The internet-facing AIG ALB is the only inbound
  internet path. It terminates TLS and forwards to AIG instances in private subnets.
- **DLPoD is fully internal.** The DLPoD ALB is internal-only (private subnets). DLPoD instances
  are reachable only from the DLPoD ALB on port 443 — nothing else in the stack can connect to
  them.
- **Outbound internet via NAT Gateway only.** All instance outbound internet traffic traverses
  the single NAT Gateway. There is no direct internet gateway route to private subnets. An S3
  Gateway Endpoint routes S3 traffic via the AWS backbone (no NAT Gateway traversal). The
  Netskope destinations DLPoD must reach (and the advice against TLS-intercepting that traffic)
  are listed in [ARCHITECTURE.md — DLPoD tier](ARCHITECTURE.md#dlp-on-demand-dlpod--internal-inspection-tier).
- **AIG → DLPoD via private DNS.** AIG instances resolve `dlp.aigw.internal` via a Route 53
  private hosted zone — this alias always resolves to the DLPoD internal ALB, never to a public
  address. DLPoD instances use the Route 53 Resolver (`169.254.169.253`) set in `bootstrap.json`.
- **IMDSv2 enforced.** All three launch templates set `HttpTokens: required`, so instance
  credentials cannot be retrieved with unauthenticated IMDSv1 requests.

---

## IAM Roles and Permissions

### Role Summary

| Role | Assumed by | Purpose |
|---|---|---|
| `<stack>-certgen-role` | `lambda.amazonaws.com` | Cert generator custom resource — self-signed cert generation (DLPoD ALB, and AIG ALB when auto-generated) |
| `<stack>-gateway-role` | `ec2.amazonaws.com` | AIG instance profile — read the bootstrap secret; CloudWatch Agent |
| `<stack>-aig-activation-role` | `lambda.amazonaws.com` | Activation Lambda — AIG lifecycle management (enroll / deregister) |
| `<stack>-aig-lifecycle-sns-role` | `autoscaling.amazonaws.com` | AIG lifecycle event delivery to SNS |
| `<stack>-dlpod-role` | `ec2.amazonaws.com` | DLPoD instance profile (CloudWatch Agent only) |
| `<stack>-dlpod-bootstrap-builder-role` | `lambda.amazonaws.com` | Assemble DLPoD `bootstrap.json` UserData at stack create/update |
| `<stack>-dlpod-readiness-role` | `lambda.amazonaws.com` | DLPoD readiness gate — poll the DLPoD ALB target group before the AIG ASG is created |
| `<stack>-guardrails-role` *(Guardrails only)* | `ec2.amazonaws.com` | Guardrails instance profile (S3 image download, SSM Session Manager, CloudWatch) |

The optional `GuardrailsReadinessGate` is a CloudFormation WaitCondition signalled from instance
UserData; it needs no IAM role of its own.

### Key Principle: AIG Instances Never Hold API Credentials

AIG instances have an IAM role (`<stack>-gateway-role`) that allows only:
1. `secretsmanager:GetSecretValue` on the specific bootstrap secret ARN
2. The AWS-managed `CloudWatchAgentServerPolicy` (metrics and logs)

The Netskope API token is in a separate secret (`<stack>-netskope-credentials`) that the AIG
instance role has **no access to**. Only the Activation Lambda reads it.

DLPoD instances (`<stack>-dlpod-role`) have **no** Secrets Manager or SSM permissions at all.

### Role Permissions Detail

Statement `Sid`s are given where the template defines them so a reviewer can `grep` the template.

**`<stack>-certgen-role` (cert generator custom resource Lambda)**
- `WriteLogs` — `logs:CreateLogStream`, `logs:PutLogEvents` on `/aws/lambda/<stack>-certgen` only
- `ImportCert` — `acm:ImportCertificate`, `acm:DeleteCertificate`, `acm:AddTagsToCertificate` on `Resource: "*"` (ACM import has no pre-existing ARN to scope to)
- `WriteCertParam` — `ssm:PutParameter` on `/<stack>/*` (`/<stack>/dlpod-cert`, `/<stack>/aig-cert`)
- `WriteCertKeySecret` — `secretsmanager:PutSecretValue` on `<stack>-dlpod-cert-key`

**`<stack>-gateway-role` (AIG instance profile)**
- `ReadBootstrapSecret` — `secretsmanager:GetSecretValue` on `<stack>-aig-bootstrap` only (ARN-scoped)
- `CloudWatchAgentServerPolicy` (AWS managed) — CloudWatch metrics and logs

**`<stack>-aig-activation-role` (Activation Lambda)**
- `WriteLogs` — `logs:CreateLogStream`, `logs:PutLogEvents` on `/aws/lambda/<stack>-aig-activation` only
- `GetNetskopeSecret` — `secretsmanager:GetSecretValue` on `<stack>-netskope-credentials` (reads API token)
- `WriteBootstrapSecret` — `secretsmanager:PutSecretValue` on `<stack>-aig-bootstrap` (writes enrollment token + DLP / Guardrails block)
- `ReadDlpodCert` — `ssm:GetParameter` on `/<stack>/dlpod-cert` (reads DLPoD CA cert for the bootstrap secret)
- `ApplianceIdParam` — `ssm:PutParameter`, `ssm:GetParameter`, `ssm:DeleteParameter` on `/aig/<stack>/*` (appliance ID tracking)
- `DescribeInstances` — `ec2:DescribeInstances` on `Resource: "*"` (looks up the launching instance's private IP; EC2 describe calls cannot be ARN-scoped)
- `CompleteLifecycle` — `autoscaling:CompleteLifecycleAction` on `<stack>-aig-asg` only

**`<stack>-aig-lifecycle-sns-role` (Auto Scaling → SNS)**
- `sns:Publish` on the `<stack>-aig-lifecycle` topic only

**`<stack>-dlpod-role` (DLPoD instance profile)**
- `CloudWatchAgentServerPolicy` (AWS managed) — nothing else

**`<stack>-dlpod-bootstrap-builder-role` (DLPoD bootstrap builder custom resource Lambda)**
- `WriteLogs` — `logs:CreateLogStream`, `logs:PutLogEvents` on `/aws/lambda/<stack>-dlpod-bootstrap-builder` only
- `ReadCertKeySecret` — `secretsmanager:GetSecretValue` on `<stack>-dlpod-cert-key` (leaf cert, leaf key, CA cert)
- `ReadCredentialsSecret` — `secretsmanager:GetSecretValue` on `<stack>-dlpod-credentials` (license key)

**`<stack>-dlpod-readiness-role` (DLPoD readiness gate custom resource Lambda)**
- `WriteLogs` — `logs:CreateLogStream`, `logs:PutLogEvents` on `/aws/lambda/<stack>-dlpod-readiness` only
- `DescribeTargetHealth` — `elasticloadbalancing:DescribeTargetHealth` on `Resource: "*"` (read-only; describe calls cannot be ARN-scoped). Used for the DLPoD target group only — the Guardrails gate is a WaitCondition and does not invoke this Lambda

**`<stack>-guardrails-role` (Guardrails instance profile — only when deployed)**
- `s3:GetObject` on the single object `arn:aws:s3:::<GuardrailsImageS3Bucket>/<GuardrailsImageS3Key>`
- `s3:ListBucket` on the bucket, with `s3:prefix` limited to the image key
- `AmazonSSMManagedInstanceCore` (AWS managed) — Session Manager access for container diagnostics
- `CloudWatchAgentServerPolicy` (AWS managed)

---

## Secret and Credential Management

### What's Stored and Where

Four Secrets Manager secrets and three SSM Parameter Store paths are created or written by the
stack. Payload shapes are shown below the table.

| Name | Type | Contents | Who writes | Who reads | Lifecycle |
|---|---|---|---|---|---|
| `<stack>-netskope-credentials` | Secrets Manager | Netskope tenant URL + API token | CloudFormation (from `NoEcho` parameter) | Activation Lambda only | Created at stack creation; deleted at teardown |
| `<stack>-aig-bootstrap` | Secrets Manager | Enrollment token (per launch), DLP host + CA cert, Guardrails host (optional) | Activation Lambda (full overwrite at every AIG launch) | AIG instances at boot | Created at stack creation with a placeholder value; deleted at teardown |
| `<stack>-dlpod-credentials` | Secrets Manager | DLPoD license key | CloudFormation (from `NoEcho` parameter) | DLPoD bootstrap builder Lambda only (stack create/update) | Created at stack creation; deleted at teardown |
| `<stack>-dlpod-cert-key` | Secrets Manager | DLPoD CA cert, leaf cert, leaf private key (PEM) | Cert generator Lambda | DLPoD bootstrap builder Lambda only | Created at stack creation with a placeholder value; deleted at teardown |
| `/<stack>/dlpod-cert` | SSM Parameter (`String`) | PEM-encoded DLPoD CA certificate (public material, 365-day validity) | Cert generator Lambda | Activation Lambda at every AIG launch | Written at stack creation; deleted at teardown |
| `/<stack>/aig-cert` | SSM Parameter (`String`) | PEM-encoded AIG ALB self-signed cert — only when `AcmCertificateArn` is empty | Cert generator Lambda | Operators (to distribute the cert to clients) | Written at stack creation; deleted at teardown |
| `/aig/<stack>/<instance-id>` | SSM Parameter (`String`) | AIG appliance ID in the Netskope tenant | Activation Lambda at launch | Activation Lambda at termination | Written at instance launch; deleted at instance termination |

`<stack>-netskope-credentials`:

```json
{"tenant_url": "https://<tenant>.goskope.com", "api_token": "..."}
```

`<stack>-aig-bootstrap` (the `ai_guardrails` block is present only when the Guardrails tier is deployed):

```json
{
  "bootstrap": true,
  "enrollment_token": "...",
  "dlp": {"host": "https://dlp.aigw.internal", "certificate": "<CA PEM>"},
  "ai_guardrails": {"host": "http://guardrails.aigw.internal:8080/invocations"}
}
```

`<stack>-dlpod-credentials`:

```json
{"license_key": "..."}
```

`<stack>-dlpod-cert-key`:

```json
{"ca_cert_pem": "...", "leaf_cert_pem": "...", "leaf_key_pem": "..."}
```

### Credential Flow

**AIG (per instance launch):**
```
1. User provides API token as NoEcho CloudFormation parameter
2. CloudFormation creates <stack>-netskope-credentials in Secrets Manager
3. ASG launches AIG instance → lifecycle hook → Activation Lambda fires
4. Activation Lambda reads API token from Secrets Manager (encrypted in transit, AWS SDK TLS)
5. Activation Lambda calls Netskope REST API → receives enrollment token (exists in Lambda memory only)
6. Activation Lambda reads the DLPoD CA cert from SSM and writes enrollment token + DLP block
   to <stack>-aig-bootstrap (separate secret)
7. AIG instance reads bootstrap secret at boot over HTTPS → self-enrolls
8. Enrollment token is consumed; it is not persisted beyond the bootstrap secret write
```

The API token (`<stack>-netskope-credentials`) and the enrollment token (`<stack>-aig-bootstrap`)
are in separate secrets. Compromise of the bootstrap secret does not expose the API token.

**DLPoD (once, at stack creation or update):**
```
1. User provides license key as NoEcho CloudFormation parameter
2. CloudFormation creates <stack>-dlpod-credentials in Secrets Manager
3. CertGeneratorFunction (<stack>-certgen) generates a CA + leaf cert for dlp.aigw.internal,
   imports the leaf to ACM, writes the CA PEM to SSM /<stack>/dlpod-cert and the full
   CA + leaf + private key to <stack>-dlpod-cert-key
4. DlpodBootstrapBuilderFunction (<stack>-dlpod-bootstrap-builder) reads both secrets and
   assembles bootstrap.json (TLS cert + key, license key, DNS 169.254.169.253, persona)
5. The base64-encoded bootstrap.json becomes the DlpodLaunchTemplate UserData
6. Every DLPoD instance's nsbootstrap.service applies bootstrap.json at first boot —
   the instance never calls Secrets Manager or SSM
```

Because the license key and the DLPoD leaf private key are embedded in the launch template
UserData, anyone with `ec2:DescribeLaunchTemplateVersions` on the account (or code running on the
instance, via IMDSv2) can read them. See Known Limitations.

---

## Encryption

Every network hop in the stack except the optional AIG → Guardrails path is TLS, and every EBS
root volume is encrypted; the tables below list each path and each storage location.

### In Transit

| Path | Protocol | Notes |
|---|---|---|
| Internet → AIG ALB | TLS (ALB default security policy) | Certificate from ACM (auto-generated self-signed or user-provided). No `SslPolicy` is set on the listener; add one to enforce TLS 1.2+ only |
| AIG ALB → AIG instances | TLS (HTTPS:443) | ALB health checks and traffic forwarding |
| AIG instances → DLPoD ALB | TLS (HTTPS:443) | Stack-generated leaf cert signed by a stack-generated CA; AIG trusts the CA via `dlp.certificate` in the bootstrap secret |
| DLPoD ALB → DLPoD instances | TLS (HTTPS:443) | DLPoD serves the same leaf cert + key delivered in `bootstrap.json` |
| DLPoD instances → Netskope management plane | TLS (HTTPS:443) | Outbound via NAT Gateway after licensing; Netskope advises against intercepting this TLS |
| AIG instances → Guardrails ALB → Guardrails instances *(if deployed)* | HTTP (`GuardrailsContainerPort`) | Plain HTTP inside private subnets — see Known Limitations |
| Guardrails instances → S3 *(if deployed)* | TLS (HTTPS:443) | Image tarball download via the S3 Gateway Endpoint |
| Lambda → Secrets Manager / SSM / ACM / ELBv2 | TLS (AWS SDK) | All AWS SDK calls use TLS |
| Lambda → Netskope REST API | TLS (HTTPS:443) | AIG enrollment and deregistration |

No component of the stack uses SSH. Guardrails instances are reachable for diagnostics only via
SSM Session Manager (`AmazonSSMManagedInstanceCore`), which is IAM-authenticated and logged in
CloudTrail; AIG and DLPoD instance roles do not include Session Manager.

### At Rest

| Resource | Encryption |
|---|---|
| Secrets Manager secrets | AES-256, AWS-managed KMS key (`aws/secretsmanager`) |
| SSM Parameter Store parameters | `String` type — hold only public certificate PEMs and appliance IDs; no private keys or credentials are stored in SSM |
| EBS root volumes (AIG 400 GB, DLPoD 351 GB, Guardrails 100 GB; all gp3) | `Encrypted: true` set explicitly in all three launch templates (AWS-managed EBS key unless the account default key is customer-managed) |
| Guardrails local NVMe instance store *(if deployed)* | Not covered by the EBS statement above. Holds only the `aisecurity-llm.tgz` tarball (deleted after `docker load`) and Docker's `data-root` (container image layers) — no customer data or credentials. Instance-store volumes are hardware-encrypted by AWS and their contents are lost on stop or terminate |
| CloudWatch Logs | Encrypted at rest by default (AWS-managed) |
| Launch template UserData (DLPoD) | Stored by EC2; not separately encrypted — contains the DLPoD TLS key and license key (see Known Limitations) |

---

## CloudFormation Security Practices

**Practices implemented in the template:**

- **`NoEcho: true` on all sensitive parameters** (`NetskopeApiToken`, `DlpodLicenseKey`) — values
  are never shown in CloudFormation events, stack output, or the console.
- **Resource-scoped IAM policies** — Secrets Manager, SSM, SNS, and Auto Scaling access is scoped
  to specific resource ARNs constructed with `!Ref` / `!Sub`. The only `Resource: "*"` grants are
  on actions that cannot be ARN-scoped (`ec2:DescribeInstances`,
  `elasticloadbalancing:DescribeTargetHealth`, `acm:ImportCertificate` and its companions).
- **No secrets in AIG user data** — AIG instance user data contains only the bootstrap secret
  *name* (`{"bootstrap_secret": "<stack>-aig-bootstrap"}`). The instance reads the value from
  Secrets Manager at boot using its IAM role.
- **No secrets in Lambda environment variables** — The Activation Lambda's environment holds only
  ARNs, parameter names, and host names. Credentials are retrieved at runtime from Secrets Manager
  using the execution role. The other three Lambdas have no environment variables at all.
- **All Lambda code is inline** — Every function uses `ZipFile` code embedded in the template. No
  external Lambda artifacts, layers, or S3 code buckets are fetched at deploy time, so the reviewed
  template is the complete supply chain. (An S3 bucket is still used to *host the template itself*
  because it exceeds the 51 KB `--template-body` limit.)
- **Separate secrets for separate purposes** — API credentials (`<stack>-netskope-credentials`),
  bootstrap data (`<stack>-aig-bootstrap`), license key (`<stack>-dlpod-credentials`), and the
  DLPoD TLS key material (`<stack>-dlpod-cert-key`) are in separate Secrets Manager secrets with
  separate, role-specific access.
- **IMDSv2 required** — All launch templates set `MetadataOptions.HttpTokens: required`.
- **Conditions prevent unnecessary resource creation** — `UseAutoGeneratedAigCert` skips the AIG
  cert generation when a user-provided ACM ARN is given; `DeployGuardrails` creates the GPU tier
  (role, SGs, ALB, ASG, WaitCondition) only when `GuardrailsImageS3Bucket` is set, and a template
  `Rules` assertion requires `GuardrailsAmiId` alongside it.

**Caveats to be aware of:**

- **AIG ALB uses a self-signed certificate by default** — The auto-generated cert has
  `aig.aigw.internal` as its CN/SAN. API clients must be configured to trust it or skip TLS
  verification. For production deployments with external clients, provide a trusted ACM certificate
  via `AcmCertificateArn` — see [DEPLOYMENT.md — Parameters](DEPLOYMENT.md#parameters).
- **DLPoD configuration travels in UserData** — The DLPoD `bootstrap.json` (TLS private key and
  license key) is base64-encoded, not encrypted, in the launch template. Restrict
  `ec2:DescribeLaunchTemplateVersions` and `ec2:DescribeInstanceAttribute` (userData) to operators.

---

## Audit and Visibility

### CloudWatch Log Groups

The stack creates four Lambda log groups (`/aws/lambda/<stack>-certgen`, `-dlpod-bootstrap-builder`,
`-aig-activation` at 30-day retention; `-dlpod-readiness` at 7-day retention). None of them ever
logs a secret value, certificate private key, or license key. The contents of each group are
described in [OPERATIONS.md — Log Groups](OPERATIONS.md#log-groups).

DLPoD appliances do not write CloudWatch logs from the stack's perspective; `nsbootstrap.service`
status is observable through DLPoD ALB target health or on the appliance per the Netskope DLP On
Demand documentation. Guardrails instances write UserData output to `/var/log/user-data.log` on
the instance (readable via SSM Session Manager), not to CloudWatch.

### Netskope Audit Log

All AIG activity — requests, responses, blocked events, policy decisions — is logged to the
Netskope management plane. Access logs from the Netskope UI under
**Analytics → SkopeIT → AI Gateway** or via the Netskope Events API.

### What to Monitor

- Lambda function errors → CloudWatch Metrics → filter on `Errors` for each function
- AIG enrollment failures → check `/aws/lambda/<stack>-aig-activation` for `Traceback` / `ABANDON` lines
- DLPoD bootstrap failures → DLPoD ALB target health (`aws elbv2 describe-target-health`); a target
  that never becomes healthy within the 30-minute ASG grace period indicates `nsbootstrap` did not
  complete (bad license key, no outbound path) — see [TROUBLESHOOTING.md](TROUBLESHOOTING.md#dlp-on-demand-issues)
- Stack rollback at `DlpodReadinessGate` → `describe-stack-events` and
  `/aws/lambda/<stack>-dlpod-readiness`
- Stack rollback at `GuardrailsReadinessGate` → `describe-stack-events` (the WaitCondition records
  the `Reason` sent by UserData, e.g. `Container not healthy after 15 min`) and
  `/var/log/user-data.log` on the Guardrails instance via SSM Session Manager. This gate does not
  use the readiness Lambda, so `/aws/lambda/<stack>-dlpod-readiness` has nothing about it
- ALB target health → `aws elbv2 describe-target-health` — unhealthy targets indicate enrollment/bootstrap problems
- CloudTrail → `secretsmanager:GetSecretValue` on `<stack>-netskope-credentials` from any principal
  other than `<stack>-aig-activation-role`; `ec2:DescribeLaunchTemplateVersions` on `<stack>-dlpod-lt`;
  `ssm:StartSession` against Guardrails instances

---

## Known Limitations and Accepted Risks

**DLPoD TLS private key and license key are in launch template UserData**
- *Detail:* `bootstrap.json` (leaf cert + private key for `dlp.aigw.internal`, DLPoD license key)
  is base64-encoded into `<stack>-dlpod-lt` UserData so `nsbootstrap.service` can apply it at first
  boot with no credentials on the instance. UserData is readable by any principal with
  `ec2:DescribeLaunchTemplateVersions` / `ec2:DescribeInstanceAttribute`, and by code on the
  instance via IMDSv2.
- *Mitigation:* The key is stack-generated, valid 365 days, and only ever trusted by this stack's
  AIG instances for `dlp.aigw.internal` (an internal-only ALB). Restrict launch-template read
  permissions to operators; a stack update that re-runs `DlpodAlbCertificate` regenerates the
  hierarchy.

**AIG → Guardrails traffic is plain HTTP**
- *Detail:* The AIG bootstrap `ai_guardrails` block carries a host only (no certificate field), so
  the Guardrails internal ALB listens on HTTP. Prompts and responses under inspection traverse this
  hop unencrypted.
- *Mitigation:* Path is confined to private subnets and restricted by security group to AIG
  instances → Guardrails ALB → Guardrails instances; no internet or cross-VPC exposure. If a
  future AIG build accepts `ai_guardrails.certificate`, switch the listener to HTTPS using the
  existing `Custom::AlbCertificate` pattern.

**Guardrails container health does not drive instance replacement**
- *Detail:* `GuardrailsAutoScalingGroup` uses `HealthCheckType: EC2`. A running instance whose
  container has stopped answering `/ping` stays in service in the ASG (the ALB stops routing to it,
  but it is never replaced automatically).
- *Mitigation:* Alarm on `UnHealthyHostCount` for the Guardrails target group and replace the
  instance manually — see
  [OPERATIONS.md — AI Guardrails](OPERATIONS.md#ai-guardrails-only-when-guardrailsimages3bucket-was-set).

**Auto-generated AIG ALB cert is self-signed**
- *Detail:* CN/SAN is `aig.aigw.internal`, which does not resolve publicly. Clients must trust the
  cert or skip TLS verification.
- *Mitigation:* Provide a valid ACM certificate via `AcmCertificateArn` for production deployments
  where clients require trusted TLS.

**Shared AIG bootstrap secret**
- *Detail:* Every AIG launch overwrites `<stack>-aig-bootstrap` with that instance's enrollment
  token; two AIG instances launching concurrently can read each other's token.
- *Mitigation:* Scale AIG one instance at a time (the CPU scale-out policy adds +1 per alarm). The
  DLP / Guardrails blocks are identical for all instances and unaffected.

**Stack-generated certificates expire after 365 days**
- *Detail:* The DLPoD CA + leaf (and the auto-generated AIG ALB cert) are valid for one year from
  stack creation. AIG → DLPoD TLS fails after expiry.
- *Mitigation:* Plan a stack update that re-runs `DlpodAlbCertificate` and an instance refresh
  before expiry — see [OPERATIONS.md — Certificate Renewal](OPERATIONS.md#certificate-renewal).

**Secrets Manager secrets deleted on stack teardown**
- *Detail:* Deleting the stack permanently deletes all Secrets Manager secrets including API
  credentials.
- *Mitigation:* If you need to preserve credentials, add a `DeletionPolicy: Retain` to the secret
  resources before deploying, or back up secret values before teardown.

**Guardrails instance store is ephemeral and unencrypted by the EBS setting**
- *Detail:* The image tarball and Docker layers live on the local NVMe instance store, which is
  outside the `Encrypted: true` EBS configuration (see [At Rest](#at-rest)).
- *Mitigation:* No customer data or credentials are written there; AWS encrypts instance-store
  volumes in hardware, and the contents are discarded on stop/terminate. Every launch re-downloads
  the tarball from S3 via the instance role's single-object `s3:GetObject` grant.
