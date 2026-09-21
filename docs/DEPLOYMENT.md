# AI Gateway + DLP On Demand — Deployment Guide

`templates/gateway-combined.yaml` deploys Netskope AI Gateway (AIG) and DLP On Demand (DLPoD)
together in a single CloudFormation stack, with an optional GPU-backed AI Guardrails tier. A new
VPC is created — no pre-existing networking is required. DLPoD configures itself automatically via
`nsbootstrap.service` at first boot using EC2 UserData. AIG enrolls via native Secrets Manager
bootstrap: at each AIG launch the Activation Lambda registers the appliance with Netskope and
writes the enrollment token plus the DLPoD certificate and endpoint (and the AI Guardrails
endpoint, when deployed) into the bootstrap secret before the instance is released to boot.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Preflight Checks](#preflight-checks)
- [Parameters](#parameters)
- [Deploy](#deploy)
- [Startup Ordering](#startup-ordering)
- [Verify Deployment](#verify-deployment)
- [Stack Outputs](#stack-outputs)
- [Update](#update)
- [Teardown](#teardown)

---

## Prerequisites

Complete the four required items below before deploying. The ACM certificate and AI Guardrails
items are optional.

The shell examples throughout this guide use two variables — set them once:

```bash
STACK=<stack-name>
REGION=<region>
```

### AI Gateway AMI

- [ ] Subscribe to the AI Gateway product in [AWS Marketplace](https://aws.amazon.com/marketplace)
  (search "Netskope AI Gateway")

After subscribing, the AMI is available in your AWS account in every region where the product is
offered.

> **Region:** Only the template *default* AMI ID is region-specific. The default
> `ami-0a66805d7fb085df4` is AI Gateway v1.7.54 in **us-west-1**. For any other region, look up
> the subscribed AMI ID and pass it as `GatewayAmiId`.

Verify your subscribed AMI before deploying:
```bash
aws ec2 describe-images \
  --filters 'Name=name,Values=*Netskope AI Gateway*' \
  --query 'sort_by(Images, &CreationDate)[-1].[ImageId,Name,State]' \
  --output table --region $REGION
```

### DLP On Demand AMI

The DLPoD image is **not** an AWS Marketplace product. Netskope shares it privately to your AWS
account.

- [ ] Share the DLP On Demand AMI to your AWS account from the Netskope console:
  **Security Cloud Platform → On-Premises Infrastructure → Setup DLP On Demand → AWS → Share
  Image** — enter your AWS account ID and choose the region. See the
  [Netskope DLP On Demand configuration guide](https://docs.netskope.com/en/dlpondemandconfig)
  for full instructions.

After sharing, the AMI appears in your account under **EC2 → AMIs → Private images** in the
region you chose at share time.

> **Warning:** The template default `ami-0973780ab75c2fb28` (DLP On Demand, us-west-1) launches
> only if that exact image has been shared to the deploying account in us-west-1. If it has not,
> stack creation fails when the DLPoD Auto Scaling Group tries to launch. The region is chosen when
> the image is shared, not by Marketplace availability — for any other region, look up the shared
> AMI ID and pass it as `DlpodAmiId`.

Look up the shared AMI ID. The name filter is indicative only — the shared image name may differ,
so the `is-public=false` filter is what narrows the result to privately shared images:
```bash
aws ec2 describe-images \
  --filters 'Name=name,Values=*Netskope DLP*' Name=is-public,Values=false \
  --query 'sort_by(Images, &CreationDate)[-1].[ImageId,Name,State]' \
  --output table --region $REGION
```

### Netskope Credentials

- [ ] **Tenant URL** — your Netskope tenant URL, e.g. `https://tenant.goskope.com`

- [ ] **RBAC v3 API token** — service account token with the `AIG Administrator` role:
  1. **Settings → Administration → Administrators & Roles → Roles** — create a role with AI Gateway /
     On-Premises Infrastructure permissions
  2. **Settings → Administration → Administrators & Roles → Administrators** — add a Service Account,
     assign the role, and copy the token (displayed once)

  > Existing REST API v2 tokens continue to work until expiry but cannot be renewed. New
  > deployments should use RBAC v3 service accounts.

- [ ] **DLP On Demand license key** — in your Netskope tenant:
  **Settings → Security Cloud Platform → On-Premises Infrastructure**

**Environment variables.** Exporting the credentials keeps them out of the conversation and your
shell history; the deploy commands in this guide reference them by name.

| Environment variable | Template parameter | Notes |
|---|---|---|
| `NETSKOPE_TENANT_URL` | `NetskopeTenantUrl` | Tenant URL only, e.g. `https://tenant.goskope.com`. If you copied a `NETSKOPE_SERVER_URL` that ends in `/api/v2`, strip that suffix — the template appends the API path itself. |
| `NETSKOPE_API_TOKEN` | `NetskopeApiToken` | RBAC v3 service account token, used as-is. A legacy REST API v2 `NETSKOPE_API_KEY` is a different credential and is not accepted. |
| `DLPOD_LICENSE_KEY` | `DlpodLicenseKey` | DLP On Demand license key, used as-is. |

```bash
export NETSKOPE_TENANT_URL=https://tenant.goskope.com
export NETSKOPE_API_TOKEN=<token>
export DLPOD_LICENSE_KEY=<license-key>
```

### ACM Certificate (optional)

The `AcmCertificateArn` parameter is optional. Leave it empty and the stack auto-generates a
self-signed certificate for the internet-facing AIG ALB (consistent with how the DLPoD internal
ALB certificate is handled). The generated cert uses `aig.aigw.internal` as its CN/SAN and is
imported to ACM by the stack.

> **Self-signed cert limitations:** API clients must be configured to trust the cert or disable
> TLS certificate verification. Browsers will show a security warning. For deployments where
> clients cannot be configured to skip verification, provide an ACM certificate ARN.

**Option A — auto-generate (default):** Omit `AcmCertificateArn` from the deploy command.

**Option B — bring your own cert:** Provide an ACM certificate ARN with
`extendedKeyUsage=serverAuth`. For public-facing deployments with a custom domain, use an
ACM-issued certificate with DNS validation via Route 53:

```bash
aws acm request-certificate \
  --domain-name <your-domain> \
  --validation-method DNS \
  --region $REGION
# Complete DNS validation in Route 53, then use the certificate ARN
```

For testing without a public domain, import a self-signed cert:
```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout key.pem -out cert.pem -days 365 \
  -subj "/CN=aigw.example.internal" \
  -addext 'subjectAltName=DNS:aigw.example.internal' \
  -addext 'extendedKeyUsage=serverAuth,clientAuth'

aws acm import-certificate \
  --certificate fileb://cert.pem \
  --private-key fileb://key.pem \
  --region $REGION
# Use the output CertificateArn as AcmCertificateArn
```

### AI Guardrails Prerequisites (optional)

Skip this section if you are not deploying the AI Guardrails tier. The tier is enabled by setting
`GuardrailsImageS3Bucket`; see [AI Guardrails Parameters](#ai-guardrails-parameters-optional).

- [ ] **Guardrails Docker image in S3** — upload `aisecurity-llm.tgz` (supplied by Netskope; a
  `docker save` archive) to an S3 bucket in the same region as the stack. At first boot each
  Guardrails instance downloads it, runs `docker load`, and starts the image tag reported by
  `docker load`.
  ```bash
  aws s3 mb s3://<guardrails-bucket> --region $REGION          # skip if it already exists
  aws s3 cp aisecurity-llm.tgz s3://<guardrails-bucket>/aisecurity-llm.tgz --region $REGION
  ```

- [ ] **Deep Learning Base GPU AMI** — the AMI must ship the NVIDIA driver, Docker and the NVIDIA
  Container Toolkit. Find the latest AWS Deep Learning Base GPU AMI (Ubuntu 22.04) for your region:
  ```bash
  aws ec2 describe-images --owners amazon --region $REGION \
    --filters "Name=name,Values=Deep Learning Base OSS Nvidia Driver GPU AMI (Ubuntu 22.04)*" \
    --query 'sort_by(Images,&CreationDate)[-1].[ImageId,Name]' --output text
  ```

- [ ] **GPU instance quota** — AWS Console → Service Quotas → Amazon EC2 → search
  "Running On-Demand G and VT instances". Minimum **4 vCPU** for `g4dn.xlarge` (default).

- [ ] **Instance type with local NVMe storage** — `GuardrailsInstanceType` is restricted to
  `g4dn`/`g5` types that ship local NVMe instance storage. UserData formats and mounts it and
  places the image tarball and the Docker data-root there; on EBS-only types boot is significantly
  slower and can exceed the 15-minute UserData budget for the readiness signal. Keep the default
  unless you have a reason to change it.

### AWS Permissions

- [ ] The deploying IAM principal has `CAPABILITY_NAMED_IAM` and the required service permissions.

This is the canonical copy of the deployer policy; other documents link here.

<details>
<summary>IAM policy JSON (click to expand)</summary>

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudFormation",
      "Effect": "Allow",
      "Action": "cloudformation:*",
      "Resource": "*"
    },
    {
      "Sid": "EC2",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    },
    {
      "Sid": "ELB",
      "Effect": "Allow",
      "Action": "elasticloadbalancing:*",
      "Resource": "*"
    },
    {
      "Sid": "AutoScaling",
      "Effect": "Allow",
      "Action": "autoscaling:*",
      "Resource": "*"
    },
    {
      "Sid": "Lambda",
      "Effect": "Allow",
      "Action": "lambda:*",
      "Resource": "*"
    },
    {
      "Sid": "IAM",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:GetRole",
        "iam:PutRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:AttachRolePolicy",
        "iam:DetachRolePolicy",
        "iam:PassRole",
        "iam:TagRole",
        "iam:UntagRole",
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:GetInstanceProfile",
        "iam:AddRoleToInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SecretsManager",
      "Effect": "Allow",
      "Action": "secretsmanager:*",
      "Resource": "*"
    },
    {
      "Sid": "SSM",
      "Effect": "Allow",
      "Action": [
        "ssm:PutParameter",
        "ssm:GetParameter",
        "ssm:DeleteParameter",
        "ssm:AddTagsToResource"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SNS",
      "Effect": "Allow",
      "Action": "sns:*",
      "Resource": "*"
    },
    {
      "Sid": "Route53",
      "Effect": "Allow",
      "Action": "route53:*",
      "Resource": "*"
    },
    {
      "Sid": "ACM",
      "Effect": "Allow",
      "Action": "acm:*",
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": "logs:*",
      "Resource": "*"
    },
    {
      "Sid": "S3Bucket",
      "Effect": "Allow",
      "Action": ["s3:CreateBucket", "s3:ListBucket", "s3:GetBucketLocation"],
      "Resource": "arn:aws:s3:::netskope-aigw-templates-*"
    },
    {
      "Sid": "S3Objects",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::netskope-aigw-templates-*/*"
    },
    {
      "Sid": "STS",
      "Effect": "Allow",
      "Action": "sts:GetCallerIdentity",
      "Resource": "*"
    }
  ]
}
```

> **Production hardening:** The `IAM` statement above uses `Resource: "*"`. For production
> deployments, scope it to your stack name prefix to prevent the deployer from creating roles
> outside the stack's scope:
> ```
> "Resource": [
>   "arn:aws:iam::*:role/<stack-prefix>-*",
>   "arn:aws:iam::*:instance-profile/<stack-prefix>-*"
> ]
> ```
> For example, if your stack name is `aigw-prod`, use `arn:aws:iam::*:role/aigw-prod-*`.

</details>

---

## Preflight Checks

Run these before deploying to catch common blockers early:

```bash
# Verify AWS identity and region
aws sts get-caller-identity
aws configure get region

# Verify the AI Gateway AMI is accessible in your account
aws ec2 describe-images --image-ids <gateway-ami-id> \
  --query 'Images[0].[ImageId,Name,State]' --output table --region $REGION

# Verify the DLP On Demand AMI has been shared to this account in this region.
# An error of "InvalidAMIID.NotFound" means the share from the Netskope console has not
# landed in this account/region yet (or the ID belongs to a different region).
aws ec2 describe-images --image-ids <dlpod-ami-id> \
  --query 'Images[0].[ImageId,Name,State]' --output table --region $REGION

# Verify Netskope API connectivity and token
curl -sf -o /dev/null -w "HTTP %{http_code}\n" \
  -H "Netskope-Api-Token: $NETSKOPE_API_TOKEN" \
  $NETSKOPE_TENANT_URL/api/v2/aig/appliances

# Verify the template S3 bucket exists
aws s3api head-bucket --bucket <bucket> 2>/dev/null && echo "Bucket exists" || echo "Bucket missing — run scripts/deploy-artifacts.sh $REGION"
```

---

## Parameters

Only three parameters are required (`NetskopeTenantUrl`, `NetskopeApiToken`, `DlpodLicenseKey`);
everything else has a working default. The tables below list every parameter by group.

### Netskope tenant

| Parameter | Type | Default | Required | Description |
|---|---|---|---|---|
| `NetskopeTenantUrl` | String | — | Yes | Netskope tenant URL, e.g. `https://tenant.goskope.com`. |
| `NetskopeApiToken` | String (NoEcho) | — | Yes | RBAC v3 API token with AIG Administrator role. |

### AI Gateway appliance

| Parameter | Type | Default | Required | Description |
|---|---|---|---|---|
| `GatewayAmiId` | AWS::EC2::Image::Id | `ami-0a66805d7fb085df4` (us-west-1) | No | AI Gateway AMI v1.7 or later. The default is us-west-1 only — override for other regions. |
| `InstanceType` | String | `m5.4xlarge` | No | Allowed: `m5.4xlarge`, `m6i.4xlarge`, `c5.4xlarge`. |
| `AcmCertificateArn` | String | `''` (auto-generate) | No | ACM certificate ARN for the internet-facing AIG ALB. Leave empty to auto-generate a self-signed cert. |
| `DesiredCapacity` | Number | `1` | No | Desired AIG instances (1–4). ASG min is fixed at 1, max at 4. |
| `ScaleOutCpuThreshold` | Number | `70` | No | Average CPU % that triggers AIG scale-out (+1 instance). |

### DLP On Demand appliance

| Parameter | Type | Default | Required | Description |
|---|---|---|---|---|
| `DlpodAmiId` | AWS::EC2::Image::Id | `ami-0973780ab75c2fb28` (us-west-1) | No | DLPoD AMI shared from the Netskope console. The default is us-west-1 and launches only if that image has been shared to the deploying account — override for other regions. See [DLP On Demand AMI](#dlp-on-demand-ami). |
| `DlpodInstanceType` | String | `c5a.4xlarge` | No | Allowed: `c5a.4xlarge`, `c5a.8xlarge`, `c5a.16xlarge`, `c5ad.4xlarge`, `c5ad.8xlarge`, `c5ad.16xlarge`. |
| `DlpodLicenseKey` | String (NoEcho) | — | Yes | DLP On Demand license key. |
| `DlpodDesiredCapacity` | Number | `1` | No | Desired DLPoD instances (1–4). ASG min is fixed at 1, max at 4. |

The DLPoD service name is fixed at `dlp.aigw.internal` (private hosted zone `aigw.internal`).

### AI Guardrails Parameters (optional)

The Guardrails tier is deployed only when `GuardrailsImageS3Bucket` is set. Leave it empty for the
standard AIG + DLPoD deployment. When enabled, the Activation Lambda adds an `ai_guardrails.host`
entry pointing at `http://guardrails.aigw.internal:<port>/invocations` to the AIG bootstrap
secret, and the AIG ASG is not created until the first Guardrails instance signals the
`GuardrailsReadinessGate` wait condition (see [Startup Ordering](#startup-ordering)).

The S3 upload and Deep Learning AMI lookup commands are in
[AI Guardrails Prerequisites](#ai-guardrails-prerequisites-optional).

| Parameter | Type | Default | Required | Description |
|---|---|---|---|---|
| `GuardrailsImageS3Bucket` | String | `''` (disabled) | No | S3 bucket holding the Netskope `aisecurity-llm.tgz` Docker image tarball. Must be in the same region as the stack. Empty disables the tier. |
| `GuardrailsImageS3Key` | String | `aisecurity-llm.tgz` | No | Object key of the tarball (a `docker save` archive). |
| `GuardrailsAmiId` | String | `''` | Yes, if bucket set | AWS Deep Learning Base GPU AMI (Ubuntu 22.04) — ships NVIDIA driver, Docker and NVIDIA Container Toolkit. Enforced by a template `Rules` assertion. |
| `GuardrailsInstanceType` | String | `g4dn.xlarge` | No | Allowed: `g4dn.xlarge`, `g4dn.2xlarge`, `g5.xlarge`, `g5.2xlarge`. Needs "Running On-Demand G and VT instances" quota of at least 4 vCPU per instance. Restricted to types with local NVMe instance storage; UserData places the image tarball and Docker data-root on it. Boot is significantly slower on EBS-only types. |
| `GuardrailsDesiredCapacity` | Number | `1` | No | Desired Guardrails instances (1–4). |
| `GuardrailsContainerPort` | Number | `8080` | No | Container port; also the internal ALB listener port. |
| `GuardrailsHealthCheckPath` | String | `/ping` | No | Health check path (expects HTTP 200) used by both the ALB target group and the UserData readiness poll. |

### VPC

The template creates a new VPC. Four `/24` subnets (two public, two private, across two AZs) are
derived automatically from `VpcCidr` with `Fn::Cidr`; instances use the Amazon-provided DNS
resolver (`169.254.169.253`), so no DNS or subnet parameters are needed.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `VpcCidr` | String | `10.0.0.0/16` | CIDR for the new VPC. Must be large enough for four `/24` subnets. |

---

## Deploy

**Why this step:** CloudFormation reads the template from S3 because it exceeds the 51 KB limit
for direct upload. The template URL tells CloudFormation where to find it. Once submitted,
CloudFormation provisions all resources in the correct dependency order automatically — VPC,
subnets, security groups, IAM roles, Lambda functions, and ASGs. DLPoD instances self-configure
at first boot via `nsbootstrap.service`; AIG instances self-enroll at first boot via Secrets Manager.

### Option A — AWS CLI

```bash
BUCKET=netskope-aigw-templates-<account-id>   # created by scripts/deploy-artifacts.sh $REGION

# Upload template to S3
aws s3 cp templates/gateway-combined.yaml \
  s3://$BUCKET/templates/gateway-combined.yaml --region $REGION

# Deploy
aws cloudformation create-stack \
  --stack-name $STACK \
  --template-url https://$BUCKET.s3.$REGION.amazonaws.com/templates/gateway-combined.yaml \
  --parameters \
    ParameterKey=NetskopeTenantUrl,ParameterValue=$NETSKOPE_TENANT_URL \
    ParameterKey=NetskopeApiToken,ParameterValue=$NETSKOPE_API_TOKEN \
    ParameterKey=DlpodLicenseKey,ParameterValue=$DLPOD_LICENSE_KEY \
  --tags Key=Project,Value=aigw Key=Environment,Value=prod Key=ManagedBy,Value=CloudFormation \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION

# To use a custom ACM certificate instead of the auto-generated self-signed cert:
#   add: ParameterKey=AcmCertificateArn,ParameterValue=<arn>
# To deploy the optional AI Guardrails tier:
#   add: ParameterKey=GuardrailsImageS3Bucket,ParameterValue=<guardrails-bucket> \
#        ParameterKey=GuardrailsAmiId,ParameterValue=<deep-learning-base-gpu-ami-id>
```

All other parameters take their defaults. Override `GatewayAmiId` and `DlpodAmiId` when
deploying outside us-west-1. `Project` and `Environment` are not template parameters — pass them
as `--tags` (shown above) so they propagate to all stack resources.

### Option B — AWS Console

The stack can be created entirely from the AWS Console: upload the template to the S3 bucket,
create a stack from its **Amazon S3 URL**, fill in the three required parameters, add tags, and
acknowledge the IAM capability. The step-by-step console walkthrough lives in [QUICKSTART.md — Option B — AWS Console (no CLI required)](QUICKSTART.md#option-b--aws-console-no-cli-required).

### Watch stack creation

```bash
aws cloudformation describe-stacks \
  --stack-name $STACK \
  --query 'Stacks[0].StackStatus' --output text --region $REGION
```

Stack resource creation takes approximately **12–18 minutes**. DLPoD instances launch first;
`DlpodReadinessGate` waits (up to 840 s) for every DLPoD target to be ALB-healthy, and only then
does the AIG ASG create instances. `CREATE_COMPLETE` is reported once all resources exist — AIG
enrollment finishes shortly after.

| Service | Time to load balancer healthy | What's happening |
|---|---|---|
| DLP On Demand | 5–10 min from instance launch | `nsbootstrap.service` applies `bootstrap.json` from UserData (TLS cert, license, DNS) |
| AI Gateway | 5–15 min from instance launch | Lifecycle hook → Activation Lambda registers appliance; instance reads bootstrap secret at boot and self-enrolls |
| AI Guardrails *(if deployed)* | typically under 15 min from instance launch | UserData downloads the image tarball to local NVMe, `docker load`s it, starts the container, and signals the wait condition once `/ping` returns 200 |

DLP forwarding becomes active once at least one AIG instance and one DLPoD instance are both
load balancer healthy.

---

## Startup Ordering

The template enforces this ordering so that DLPoD (and AI Guardrails, when deployed) is serving
before any AIG instance launches, and so the Activation Lambda can write a complete bootstrap
secret at each AIG launch:

```
1. CertGeneratorFunction custom resources run
   → DlpodAlbCertificate: generates a self-signed CA plus a leaf certificate for dlp.aigw.internal,
     imports the leaf to ACM, writes the CA PEM to SSM /<stack>/dlpod-cert, and writes
     CA + leaf + key to Secrets Manager <stack>-dlpod-cert-key
   → AigAlbCertificate (only when AcmCertificateArn is empty): same for aig.aigw.internal,
     CA PEM to SSM /<stack>/aig-cert
   The AIG bootstrap secret is created with the placeholder {"bootstrap": true, "enrollment_token": ""}
   — it does NOT yet contain a dlp block.

2. DlpodBootstrapPart1 / DlpodBootstrapPart2 custom resources assemble bootstrap.json
   (cert + key + license key + DNS) into the DLPoD launch template UserData
   DlpodAutoScalingGroup launches DLPoD instances:
   → nsbootstrap.service applies TLS certs, license, DNS, and persona at first boot
   → DlpodReadinessGate (Custom::DlpodReadiness, Lambda) polls the DLPoD ALB target group
     every 30 s, up to 840 s, until every target is healthy

2b. (Optional, when GuardrailsImageS3Bucket is set — runs in parallel with step 2)
   GuardrailsAutoScalingGroup launches GPU instances:
   → UserData mounts the local NVMe, downloads the tarball from S3, `docker load`s it, and
     starts the container on the configured port
   → The first instance polls its local /ping for up to 15 minutes, then signals
     GuardrailsReadinessGate (AWS::CloudFormation::WaitCondition, Timeout 3600 s, Count 1)
     with SUCCESS — or FAILURE if the container never became healthy

3. GatewayAutoScalingGroup launches AIG instances (DependsOn DlpodReadinessGate, and
   GuardrailsReadinessGate via a !Ref when Guardrails is deployed):
   → Lifecycle hook → Activation Lambda registers the appliance with the Netskope API, reads the
     DLPoD CA PEM from SSM, and writes {enrollment_token, dlp: {certificate, host}}
     (plus ai_guardrails.host when Guardrails is deployed) to the bootstrap secret
   → AIG reads the bootstrap secret at boot: enrolls + configures DLP (and Guardrails) forwarding
   → DLP forwarding is active from first AIG boot
```

---

## Verify Deployment

This is the canonical verification command set; QUICKSTART.md carries a shorter subset. Run these
after `CREATE_COMPLETE`.

### Stack status

```bash
aws cloudformation describe-stacks --stack-name $STACK \
  --query 'Stacks[0].StackStatus' --output text --region $REGION
```

Expect `CREATE_COMPLETE`.

### DLP On Demand bootstrap

```bash
# nsbootstrap progress (on the instance via SSM Session Manager)
journalctl -u nsbootstrap.service --no-pager

# ALB target health — healthy means nsbootstrap completed and HTTPS is serving
TG_ARN=$(aws elbv2 describe-target-groups \
  --query "TargetGroups[?contains(TargetGroupName,'$STACK-dlpod-tg')].TargetGroupArn" \
  --output text --region $REGION)
aws elbv2 describe-target-health --target-group-arn "$TG_ARN" \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]" \
  --output table --region $REGION
```

`healthy` = nsbootstrap completed and DLPoD is serving HTTPS on port 443.

### DLP On Demand ASG state

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-dlpod-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION
```

`InService` / `Healthy` means bootstrap completed and load balancer healthy.

### AI Gateway bootstrap secret

```bash
aws secretsmanager get-secret-value \
  --secret-id $STACK-aig-bootstrap \
  --query SecretString --output text --region $REGION
```

After the first AIG launch the secret should contain `enrollment_token`, `dlp.certificate`, and
`dlp.host` (and `ai_guardrails.host` when Guardrails is deployed).

### AI Gateway ASG state

```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-aig-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION
```

`InService` / `Healthy` means enrolled and serving HTTPS on port 443.

### AI Gateway ALB target health

```bash
TG_ARN=$(aws elbv2 describe-target-groups \
  --query "TargetGroups[?contains(TargetGroupName,'$STACK-aig-tg')].TargetGroupArn" \
  --output text --region $REGION)

aws elbv2 describe-target-health --target-group-arn "$TG_ARN" \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]" \
  --output table --region $REGION
```

`healthy` = enrolled and serving HTTPS on port 443.

### AI Guardrails target health (if deployed)

```bash
TG_ARN=$(aws elbv2 describe-target-groups \
  --query "TargetGroups[?contains(TargetGroupName,'$STACK-guardrails-tg')].TargetGroupArn" \
  --output text --region $REGION)

aws elbv2 describe-target-health --target-group-arn "$TG_ARN" \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]" \
  --output table --region $REGION
```

`healthy` = the container answers `GuardrailsHealthCheckPath` with HTTP 200.

---

## Stack Outputs

The stack exposes the values you need to connect applications and operate the deployment.
Guardrails outputs exist only when that tier is deployed; `AigAlbCertParameterName` exists only
when the AIG ALB certificate was auto-generated.

| Output | Description |
|---|---|
| `AigAlbDnsName` | AIG internet-facing ALB DNS name — create a CNAME to this value |
| `AigAutoScalingGroupName` | AIG ASG name |
| `AigBootstrapSecretName` | AIG Secrets Manager bootstrap secret name |
| `AigAlbCertParameterName` | *(auto-generated cert only)* SSM parameter containing the AIG ALB self-signed CA cert PEM |
| `AigActivationLogGroup` | Activation Lambda log group |
| `AigScaleOutAlarmName` | CloudWatch alarm that triggers AIG scale-out |
| `DlpodServiceUrl` | DLPoD private HTTPS URL (`https://dlp.aigw.internal`) |
| `DlpodCertParameterName` | SSM parameter containing the DLPoD CA certificate PEM |
| `DlpodBootstrapLogGroup` | DLPoD bootstrap builder Lambda log group |
| `GuardrailsServiceUrl` | *(Guardrails only)* Inference URL written to the bootstrap secret as `ai_guardrails.host` |
| `GuardrailsAlbDnsName` | *(Guardrails only)* Guardrails internal ALB DNS name |
| `GuardrailsAutoScalingGroupName` | *(Guardrails only)* Guardrails ASG name |
| `VpcId` | VPC ID |

**Retrieve all outputs at once:**
```bash
aws cloudformation describe-stacks --stack-name $STACK \
  --query "Stacks[0].Outputs[*].[OutputKey,OutputValue]" \
  --output table --region $REGION
```

---

## Update

Use `update-stack` with the same S3 template URL to change a parameter or roll out a new template
revision. Every parameter without a template default must be supplied on update, and any
parameter you omit reverts to its default — so pass `UsePreviousValue=true` for everything you
are not changing.

```bash
aws cloudformation update-stack \
  --stack-name $STACK \
  --template-url https://$BUCKET.s3.$REGION.amazonaws.com/templates/gateway-combined.yaml \
  --parameters \
    ParameterKey=NetskopeTenantUrl,UsePreviousValue=true \
    ParameterKey=NetskopeApiToken,UsePreviousValue=true \
    ParameterKey=DlpodLicenseKey,UsePreviousValue=true \
    ParameterKey=AcmCertificateArn,UsePreviousValue=true \
    ParameterKey=GuardrailsImageS3Bucket,UsePreviousValue=true \
    ParameterKey=GuardrailsImageS3Key,UsePreviousValue=true \
    ParameterKey=GuardrailsAmiId,UsePreviousValue=true \
    ParameterKey=<changed-parameter>,ParameterValue=<new-value> \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION
```

> **Warning:** `GuardrailsImageS3Bucket` and `GuardrailsAmiId` default to empty. If a Guardrails
> tier is deployed and you omit them from `update-stack`, CloudFormation treats the tier as
> disabled and **tears it down**. Always carry them with `UsePreviousValue=true`. Omitting
> `NetskopeTenantUrl` (no default) fails the update with a "must have values" error.

Changing `GatewayAmiId` or `DlpodAmiId` creates a new launch template version but does **not**
replace running instances — no ASG in the template has an `UpdatePolicy`. Start the rollout
manually with `aws autoscaling start-instance-refresh`; replaced AIG instances re-enroll
autonomously and replaced DLPoD instances re-run `nsbootstrap` from their UserData. See
[OPERATIONS.md — AMI Upgrade Procedure](OPERATIONS.md#ami-upgrade-procedure) for the
step-by-step process.

---

## Teardown

Deleting the stack removes everything it created; no manual cleanup is needed for stack-owned
resources.

```bash
aws cloudformation delete-stack --stack-name $STACK --region $REGION
```

Deletes all ASGs and ALBs (including the Guardrails tier if deployed), all Lambda functions, the
Route 53 private hosted zone, both auto-generated ACM certificates (the DLPoD ALB certificate and,
when `AcmCertificateArn` was left empty, the AIG ALB certificate), IAM roles, Secrets Manager
secrets, SSM parameters, and the entire VPC with subnets and NAT gateway.

Stack deletion takes approximately **8–15 minutes**, dominated by NAT gateway and VPC deletion.

> An ACM certificate passed as `AcmCertificateArn` (for the AIG ALB) is **not** deleted — it was
> imported externally and is not owned by this stack. The Guardrails image bucket and template
> bucket are likewise outside the stack and are left in place.
