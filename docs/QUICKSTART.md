# Quick Start — AI Gateway + DLP On Demand

Get Netskope AI Gateway (AIG) and DLP On Demand (DLPoD) deployed and traffic flowing in about
30–45 minutes, including the Marketplace subscription, AMI sharing, and the 12–18 minute stack
build. This guide is written for Netskope customers and sales engineers — AWS CLI experience is
helpful but not required. A console alternative is provided for deploy.

See [DEPLOYMENT.md](DEPLOYMENT.md) for the full parameter reference and advanced options.

## Table of Contents

- [What You'll Need](#what-youll-need)
- [Step 1 — Make the AMIs Available](#step-1--make-the-amis-available)
- [Step 2 — Create the Template Bucket](#step-2--create-the-template-bucket)
- [Step 3 — Deploy the Stack](#step-3--deploy-the-stack)
- [What to Expect](#what-to-expect)
- [Verify Deployment](#verify-deployment)
- [Next Steps](#next-steps)

---

## What You'll Need

Complete this checklist before starting. All five items are required.

- [ ] **AWS account** with IAM permissions to deploy CloudFormation stacks with `CAPABILITY_NAMED_IAM`.
  A minimal IAM policy is in [DEPLOYMENT.md — AWS Permissions](DEPLOYMENT.md#aws-permissions).

- [ ] **AWS CLI** installed and configured (`aws configure` or environment variables set).
  Verify with: `aws sts get-caller-identity`

- [ ] **Netskope tenant URL** — your tenant URL in the form `https://tenant.goskope.com`.
  Find this in your browser when logged into the Netskope portal.

- [ ] **Netskope RBAC v3 API token** — a service account token with the `AIG Administrator` role.

  > **Where to find it in the Netskope portal:**
  > 1. **Settings → Administration → Administrators & Roles → Roles** — create a role with
  >    AI Gateway / On-Premises Infrastructure permissions (or use an existing AIG Administrator role)
  > 2. **Settings → Administration → Administrators & Roles → Administrators** — add a Service Account,
  >    assign the role, and copy the token shown (it is displayed once only)

- [ ] **DLP On Demand license key** — your DLPoD license key.

  > **Where to find it in the Netskope portal:**
  > **Settings → Security Cloud Platform → On-Premises Infrastructure**

> **No build tools required.** All four Lambda functions in the template are inline — there is
> nothing to package, no Lambda layer, and no Docker step. The only S3 upload is the template
> itself (Step 2), because it is larger than CloudFormation's 51 KB direct-upload limit.

> **Optional — AI Guardrails:** the template can also deploy a GPU-backed AI Guardrails tier
> (`GuardrailsImageS3Bucket` + `GuardrailsAmiId`). It needs the `aisecurity-llm.tgz` tarball in an
> S3 bucket, a Deep Learning Base GPU AMI, and G-instance quota. Stack creation then also waits on
> a CloudFormation wait condition that the first Guardrails instance signals once its container
> passes `/ping` (the image is pulled from S3 onto local NVMe; allow up to 15 minutes). This guide
> leaves it disabled; see
> [DEPLOYMENT.md — AI Guardrails Prerequisites](DEPLOYMENT.md#ai-guardrails-prerequisites-optional).

---

## Step 1 — Make the AMIs Available

Both AMIs must be available in your AWS account before the stack can launch instances. They are
obtained differently: the AI Gateway AMI is an AWS Marketplace subscription; the DLP On Demand AMI
is shared privately to your account from the Netskope console.

**AI Gateway (AWS Marketplace):**
1. Go to [AWS Marketplace](https://aws.amazon.com/marketplace) and search for **Netskope AI Gateway**
2. Click **Continue to Subscribe**
3. Accept the terms and click **Accept Terms**
4. Wait for the subscription to activate (typically 1–2 minutes)

**DLP On Demand (shared from the Netskope console — not on Marketplace):**
1. In your Netskope tenant: **Security Cloud Platform → On-Premises Infrastructure →
   Setup DLP On Demand → AWS → Share Image**
2. Enter your AWS account ID and choose the region — the region is fixed at share time
3. The AMI appears in that region under **EC2 → AMIs → Private images**. See the
   [DLP On Demand configuration guide](https://docs.netskope.com/en/dlpondemandconfig) for details

> **Region note:** The AMI defaults (`GatewayAmiId` = `ami-0a66805d7fb085df4`,
> `DlpodAmiId` = `ami-0973780ab75c2fb28`) are for **us-west-1 only**. The DLPoD default launches
> only if that exact image has been shared to your account in us-west-1 — otherwise the stack
> fails at DLPoD instance launch. If deploying in a different region (or a different image was
> shared), look up the AMI IDs:
> ```bash
> REGION=<region>
>
> # AI Gateway (Marketplace)
> aws ec2 describe-images \
>   --filters 'Name=name,Values=*Netskope AI Gateway*' \
>   --query 'sort_by(Images, &CreationDate)[-1].[ImageId,Name]' \
>   --output table --region $REGION
>
> # DLP On Demand (privately shared; the name filter is indicative only)
> aws ec2 describe-images \
>   --filters 'Name=name,Values=*Netskope DLP*' Name=is-public,Values=false \
>   --query 'sort_by(Images, &CreationDate)[-1].[ImageId,Name]' \
>   --output table --region $REGION
> ```
> Pass the results as `GatewayAmiId` and `DlpodAmiId` in Step 3.

---

## Step 2 — Create the Template Bucket

**Why this step:** `templates/gateway-combined.yaml` is about 71 KB, which exceeds CloudFormation's
51 KB limit for direct template upload (`--template-body` or the console file picker). The template
must be stored in an S3 bucket **in the same region as your stack**, and CloudFormation reads it
from there. Nothing else goes in the bucket — there are no Lambda packages or layers to upload.

### Option A — Script (recommended)

```bash
REGION=<region>   # your target deployment region (same value as in Step 1)
scripts/deploy-artifacts.sh $REGION
```

This creates a bucket named `netskope-aigw-templates-<account-id>` in the target region (or
reuses it if it already exists) and prints the bucket name at the end. To use a different bucket
name: `TEMPLATE_BUCKET=<name> scripts/deploy-artifacts.sh $REGION`.

### Option B — Manual (AWS Console)

Open the [S3 Console](https://s3.console.aws.amazon.com/s3/) and click **Create bucket**.

- **Bucket name:** `netskope-aigw-templates-<your-account-id>` (replace with your 12-digit AWS account ID)
- **Region:** Select your target deployment region
- **Block Public Access:** Leave all four checkboxes enabled (default)
- All other settings: leave as defaults

Click **Create bucket**.

> Note the bucket name — you will upload the template to it in Step 3.

---

## Step 3 — Deploy the Stack

**Why this step:** Upload the template to the bucket from Step 2, then point CloudFormation at
its S3 URL. You can deploy from the AWS CLI or entirely from the AWS Console.

### Option A — AWS CLI

Export the credentials first so they stay out of your shell history and the command line:

```bash
export NETSKOPE_TENANT_URL=https://tenant.goskope.com
export NETSKOPE_API_TOKEN=<token>
export DLPOD_LICENSE_KEY=<license-key>
```

Then upload and deploy:

```bash
BUCKET=netskope-aigw-templates-<account-id>   # from Step 2
REGION=<region>                               # same region as the bucket
STACK=<stack-name>

# Upload the template to S3
aws s3 cp templates/gateway-combined.yaml \
  s3://$BUCKET/templates/gateway-combined.yaml --region $REGION

# Deploy the stack
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
```

Only these three parameters are required. All others use defaults. The `--tags` line passes
`Project`, `Environment`, and `ManagedBy` tags to all stack resources (`Project` and `Environment`
are not template parameters — use tags). The stack auto-generates a self-signed certificate for the
AIG ALB — omitting `AcmCertificateArn` is intentional. To use a custom ACM certificate,
add: `ParameterKey=AcmCertificateArn,ParameterValue=<arn>`.

Override `GatewayAmiId` and `DlpodAmiId` when deploying outside us-west-1:
`ParameterKey=GatewayAmiId,ParameterValue=<ami-id> ParameterKey=DlpodAmiId,ParameterValue=<ami-id>`.

### Option B — AWS Console (no CLI required)

This is the canonical console walkthrough; DEPLOYMENT.md links here.

**1. Upload the template to S3**

In your S3 bucket from Step 2, click **Create folder**, enter `templates`, click **Create folder**.
Open the `templates/` folder, click **Upload** → **Add files**, and select
`templates/gateway-combined.yaml` from this repository. Click **Upload**.

Your template URL will be:
```
https://netskope-aigw-templates-<account-id>.s3.<region>.amazonaws.com/templates/gateway-combined.yaml
```

**2. Open CloudFormation**

Open the [AWS CloudFormation Console](https://console.aws.amazon.com/cloudformation/), confirm
you are in the correct region (top-right corner), and click **Create stack** →
**With new resources (standard)**.

**3. Specify the template**

Select **Amazon S3 URL** and paste the template URL from above. Click **Next**.

**4. Fill in stack parameters**

Enter a stack name and fill in the three required parameters:

| Parameter | Value |
|---|---|
| Stack name | Your chosen stack name (e.g. `aigw-prod`) — used as the prefix for every resource name |
| `NetskopeTenantUrl` | `https://<tenant>.goskope.com` |
| `NetskopeApiToken` | Your RBAC v3 API token |
| `DlpodLicenseKey` | Your DLP On Demand license key |

Leave all other parameters at their defaults. In particular:

| Parameter | Default | Notes |
|---|---|---|
| `AcmCertificateArn` | *(blank)* | Leave **blank** to auto-generate a self-signed certificate |
| `GatewayAmiId` / `DlpodAmiId` | us-west-1 AMIs | Override only when deploying outside **us-west-1** (see the Step 1 region note) |
| `InstanceType` / `DlpodInstanceType` | `m5.4xlarge` / `c5a.4xlarge` | Sizing — see [ARCHITECTURE.md](ARCHITECTURE.md#instance-sizing-and-throughput) |
| `DesiredCapacity` / `DlpodDesiredCapacity` | `1` / `1` | Initial instance counts (1–4) |
| `ScaleOutCpuThreshold` | `70` | Average CPU % that adds an AIG instance |
| `VpcCidr` | `10.0.0.0/16` | New VPC CIDR; subnets are derived automatically |
| `GuardrailsImageS3Bucket` and other `Guardrails*` | *(blank / disabled)* | Leave blank to skip the optional AI Guardrails tier |

Click **Next**.

**5. Configure stack options**

Under **Tags**, add `Project`, `Environment`, and `ManagedBy` tags (e.g. `aigw`, `prod`,
`CloudFormation`). These propagate to all stack resources. No other changes required. Click **Next**.

**6. Review and submit**

On the review page, scroll to the bottom and check the box:

> **I acknowledge that AWS CloudFormation might create IAM resources with custom names.**

Click **Submit**. CloudFormation opens the stack events view — refresh to watch progress.

---

## What to Expect

Stack creation takes approximately **12–18 minutes** and is strictly ordered: DLPoD must be
healthy before the first AIG instance launches.

| Phase | Time | What's happening |
|---|---|---|
| Certificates + bootstrap | ~1 min | `<stack>-certgen` generates the DLPoD TLS hierarchy; `<stack>-dlpod-bootstrap-builder` assembles `bootstrap.json` into the DLPoD launch template UserData |
| DLP On Demand | 5–10 min from instance launch | Appliance's `nsbootstrap.service` applies the cert, license key, DNS, and persona from UserData; DLPoD ALB target becomes healthy |
| Readiness gate | until DLPoD healthy (max 840 s) | `DlpodReadinessGate` shows `CREATE_IN_PROGRESS` while it polls target health — this is normal |
| AI Gateway | 5–15 min from instance launch | Activation Lambda registers the appliance with Netskope and writes the bootstrap secret; the instance reads it at boot and self-enrolls with DLP forwarding configured |

At `CREATE_COMPLETE` the DLPoD tier is already healthy; the AIG instance may still be finishing
enrollment for a few minutes (the AIG ASG grace period is 10 minutes). DLP inspection is active as
soon as the AIG ALB target is healthy.

> **About the self-signed certificate:** The AIG ALB presents a self-signed certificate
> (`aig.aigw.internal` as CN/SAN). API clients must be configured to trust the cert or skip TLS
> verification. Browsers will show a security warning. This is expected behavior for the default
> deployment. See [DEPLOYMENT.md — ACM Certificate](DEPLOYMENT.md#acm-certificate-optional) to
> use a trusted certificate instead.

---

## Verify Deployment

Run these three checks after `CREATE_COMPLETE` (`$STACK` and `$REGION` from Step 3). The full
command set — DLPoD target health, bootstrap secret contents, per-tier ASG state — is in
[DEPLOYMENT.md — Verify Deployment](DEPLOYMENT.md#verify-deployment).

**1. Stack outputs (get the ALB DNS name and other values):**
```bash
aws cloudformation describe-stacks --stack-name $STACK \
  --query "Stacks[0].Outputs[*].[OutputKey,OutputValue]" \
  --output table --region $REGION
```

**2. AI Gateway instances in service:**
```bash
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-aig-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION
```
Look for `InService` / `Healthy`. `Pending:Wait` for up to 15 minutes after launch is normal
while the Activation Lambda registers the appliance and the instance enrolls.

**3. Quick connectivity test:**
```bash
AIG_ALB=$(aws cloudformation describe-stacks --stack-name $STACK \
  --query "Stacks[0].Outputs[?OutputKey=='AigAlbDnsName'].OutputValue" \
  --output text --region $REGION)

curl -sk -o /dev/null -w "HTTP %{http_code}\n" https://$AIG_ALB/
```
`HTTP 200` or `HTTP 401` (auth required) confirms the AIG is serving requests.

---

## Next Steps

**Point your application at the gateway:**
Create a DNS CNAME (or Route 53 alias) from your application's LLM endpoint to the
`AigAlbDnsName` stack output.

**Configure AI Gateway policy in the Netskope portal:**
After the gateway enrolls, it appears in your Netskope tenant under
**Settings → Security Cloud Platform → AI Gateway**. Configure DLP profiles, access policies,
and rate limits from there.

**Test a request:**
Send a test prompt through the gateway to verify DLP inspection is active:
```bash
curl -sk https://$AIG_ALB/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-app-token>" \
  -d '{"model": "gpt-4", "messages": [{"role": "user", "content": "Hello"}]}'
```

**Set up monitoring:**
See [OPERATIONS.md — Monitoring and Alerts](OPERATIONS.md#monitoring-and-alerts) for recommended
CloudWatch alarms and log group references.

**If something looks wrong:**
See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — the Diagnostic Commands section runs in under
2 minutes and pinpoints most issues.
