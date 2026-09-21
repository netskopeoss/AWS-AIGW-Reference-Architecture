# CLAUDE.md

Project instructions for Claude Code — deployment and operations.

For template development and modification guidelines, see [CLAUDE_DEV.md](CLAUDE_DEV.md).

---

## What This Project Does

This is an **AWS CloudFormation reference architecture for Netskope AI Gateway (AIG) and DLP On Demand (DLPoD)**.
It provisions, enrolls, and operates both services automatically — no manual steps after deployment.

The AI Gateway sits inline between applications and LLM providers (Bedrock, OpenAI, etc.),
enforcing DLP, prompt injection detection, access control, rate limiting, and audit logging.
DLP On Demand runs the content inspection locally inside the VPC so no data leaves the AWS account.

---

## Template

This repository contains a single CloudFormation template: `templates/gateway-combined.yaml`.

It deploys AIG + DLP On Demand (+ optional AI Guardrails) together in one stack,
automatically wiring the DLP certificate and endpoint into the AIG bootstrap configuration
before any instances launch. A new VPC is created — no pre-existing networking required.

> **Standalone templates** for deploying AIG or DLPoD individually are in a separate
> repository: [AWS-POV-Templates-CFT](https://github.com/jharris-ns/AWS-POV-Templates-CFT).

---

## Using Docs to Deploy or Operate

The simplest way to perform any task is to ask Claude to read the relevant document and follow it.

| Task | Instruction to give |
|---|---|
| Deploy the stack | "Read docs/DEPLOYMENT.md and deploy" |
| Quick deploy (minimal guidance) | "Read docs/QUICKSTART.md and deploy" |
| Operate a running stack | "Read docs/OPERATIONS.md" |
| Troubleshoot a failing stack | "Read docs/TROUBLESHOOTING.md" |
| Understand the architecture | "Read docs/ARCHITECTURE.md" |
| Review security posture | "Read docs/SECURITY.md" |

Providing credentials as environment variables avoids them appearing in the conversation:

```bash
export NETSKOPE_TENANT_URL=https://<tenant>.goskope.com
export NETSKOPE_API_TOKEN=<token>
export DLPOD_LICENSE_KEY=<license-key>
```

Then reference them by name: "use $NETSKOPE_API_TOKEN for the API token".

---

## Prerequisites

Before deploying, confirm:

- [ ] AWS CLI configured (`aws sts get-caller-identity` returns your account)
- [ ] IAM permissions for CloudFormation, EC2, IAM, ELB, Auto Scaling, Lambda,
  Secrets Manager, SNS, Route 53, ACM, SSM, CloudWatch — see [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md#aws-permissions)
- [ ] AI Gateway AMI subscribed in AWS Marketplace (search "Netskope AI Gateway")
- [ ] DLP On Demand AMI shared to your AWS account from the Netskope console — it is not on
  AWS Marketplace. Go to **Security Cloud Platform → On-Premises Infrastructure → Setup DLP On
  Demand → AWS → Share Image**, enter your AWS account ID and choose the region; the image then
  appears under EC2 → AMIs → Private images. See the
  [DLP On Demand config guide](https://docs.netskope.com/en/dlpondemandconfig).
- [ ] Netskope tenant URL (`https://<tenant>.goskope.com`)
- [ ] Netskope RBAC v3 API token with AIG Administrator role
- [ ] DLP On Demand license key
- [ ] *(Optional AI Guardrails)* `aisecurity-llm.tgz` image tarball uploaded to an S3 bucket in the
  target region, a Deep Learning Base GPU AMI ID for the region, and EC2 quota for
  "Running On-Demand G and VT instances" ≥ 4 vCPU — see [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md#ai-guardrails-prerequisites-optional)

> **Region:** AMI defaults are for **us-west-1 only**. For the AI Gateway, look up the Marketplace
> AMI ID for your region after subscribing and pass it as `GatewayAmiId`. For DLP On Demand, the
> region is chosen when you share the image from the Netskope console; pass the resulting AMI ID
> as `DlpodAmiId`. The default `DlpodAmiId` (`ami-0973780ab75c2fb28`) launches only if that exact
> image has been shared to the deploying account in us-west-1.

---

## Deployment (Quick Reference)

Full instructions: [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)

```bash
STACK=<stack-name>
REGION=<region>

# Step 1 — Create S3 bucket and upload template (required — exceeds 51 KB limit)
scripts/deploy-artifacts.sh $REGION
# Creates bucket netskope-aigw-templates-<account-id>
# Override bucket: TEMPLATE_BUCKET=<name> scripts/deploy-artifacts.sh $REGION
BUCKET=netskope-aigw-templates-<account-id>
aws s3 cp templates/gateway-combined.yaml \
  s3://$BUCKET/templates/gateway-combined.yaml --region $REGION

# Step 2 — Deploy
aws cloudformation create-stack \
  --stack-name $STACK \
  --template-url https://$BUCKET.s3.$REGION.amazonaws.com/templates/gateway-combined.yaml \
  --parameters \
    ParameterKey=NetskopeTenantUrl,ParameterValue=https://tenant.goskope.com \
    ParameterKey=NetskopeApiToken,ParameterValue=<token> \
    ParameterKey=DlpodLicenseKey,ParameterValue=<license-key> \
  --tags Key=Project,Value=aigw Key=Environment,Value=prod Key=ManagedBy,Value=CloudFormation \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION
# Optional AI Guardrails tier — add:
#   ParameterKey=GuardrailsImageS3Bucket,ParameterValue=<bucket-holding-aisecurity-llm.tgz> \
#   ParameterKey=GuardrailsAmiId,ParameterValue=<deep-learning-base-gpu-ami-id> \
```

Stack creation takes **12–18 minutes**. After `CREATE_COMPLETE`, both services are enrolled and
serving. Check progress:

```bash
aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query 'Stacks[0].StackStatus' --output text
```

---

## Operations Quick Reference

Full reference: [docs/OPERATIONS.md](docs/OPERATIONS.md). All commands assume
`STACK=<stack-name>` and `REGION=<region>` are set.

```bash
# --- AI Gateway (AIG) ---

# AIG instance states
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $STACK-aig-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION

# AIG enrollment / Activation Lambda logs (look for "Registered appliance")
aws logs tail /aws/lambda/$STACK-aig-activation --since 30m --region $REGION

# AIG target health
aws elbv2 describe-target-health --region $REGION --output table \
  --target-group-arn $(aws elbv2 describe-target-groups --region $REGION \
    --query "TargetGroups[?contains(TargetGroupName,'$STACK-aig')].TargetGroupArn" --output text)

# Scale AIG
aws autoscaling update-auto-scaling-group --auto-scaling-group-name $STACK-aig-asg \
  --desired-capacity <N> --region $REGION

# --- DLP On Demand (DLPoD) ---

# DLPoD instance states
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $STACK-dlpod-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION

# DLPoD bootstrap builder logs
aws logs tail /aws/lambda/$STACK-dlpod-bootstrap-builder --since 30m --region $REGION

# DLPoD readiness gate logs (stack create only)
aws logs tail /aws/lambda/$STACK-dlpod-readiness --since 30m --region $REGION

# DLPoD target health
aws elbv2 describe-target-health --region $REGION --output table \
  --target-group-arn $(aws elbv2 describe-target-groups --region $REGION \
    --query "TargetGroups[?contains(TargetGroupName,'$STACK-dlpod')].TargetGroupArn" --output text)

# Scale DLPoD
aws autoscaling update-auto-scaling-group --auto-scaling-group-name $STACK-dlpod-asg \
  --desired-capacity <N> --region $REGION

# --- AI Guardrails (only if deployed) ---

# Guardrails target health
aws elbv2 describe-target-health --region $REGION --output table \
  --target-group-arn $(aws elbv2 describe-target-groups --region $REGION \
    --query "TargetGroups[?contains(TargetGroupName,'$STACK-guardrails')].TargetGroupArn" --output text)

# Guardrails readiness gate: a CloudFormation WaitCondition signalled from the first
# instance's UserData. Check stack events for GuardrailsReadinessGate, or inspect
# the instance directly:
aws ssm start-session --target <instance-id> --region $REGION
#   then: sudo docker logs guardrails ; cat /var/log/user-data.log

# Scale Guardrails
aws autoscaling update-auto-scaling-group --auto-scaling-group-name $STACK-guardrails-asg \
  --desired-capacity <N> --region $REGION

# --- Stack ---

# Stack outputs
aws cloudformation describe-stacks --stack-name $STACK --region $REGION \
  --query "Stacks[0].Outputs[*].[OutputKey,OutputValue]" --output table

# Delete stack
aws cloudformation delete-stack --stack-name $STACK --region $REGION
```

---

## Architecture Summary

```
Internet → AIG ALB (HTTPS:443, internet-facing)
               ↓
         AI Gateway ASG (private subnets)
               ↓ inline DLP inspection
         DLPoD ALB (HTTPS:443, internal, dlp.aigw.internal)
               ↓
         DLP On Demand ASG (private subnets)

         [optional] AI Gateway ASG → Guardrails ALB (HTTP:8080, internal, guardrails.aigw.internal)
                                        ↓
                                  Guardrails GPU ASG (aisecurityllm container, private subnets)
```

**AI Guardrails (optional):** set `GuardrailsImageS3Bucket` (S3 bucket holding `aisecurity-llm.tgz`) and `GuardrailsAmiId` (Deep Learning Base GPU AMI).
The Activation Lambda then adds `ai_guardrails.host` to the AIG bootstrap secret. AIG launch is held by
`GuardrailsReadinessGate`, an `AWS::CloudFormation::WaitCondition` (60-minute timeout) that the first
Guardrails instance signals from its UserData once the local container answers `/ping` with 200 (UserData
gives up after 15 minutes and signals FAILURE). Guardrails instances place the image tarball and Docker
data-root on local NVMe instance storage, which is why `GuardrailsInstanceType` is limited to g4dn/g5 types.
Leave `GuardrailsImageS3Bucket` empty to skip the tier.

**Lifecycle automation (AIG):** ASG launch hook → SNS → Activation Lambda → registers appliance
with Netskope API → writes enrollment token to Secrets Manager → `CompleteLifecycleAction` → InService.
AIG reads the bootstrap secret at boot and self-enrolls, including DLPoD TLS config.

**DLPoD bootstrap:** `nsbootstrap.service` reads `bootstrap.json` from EC2 UserData at first boot
and applies TLS certs, license key, DNS, and persona — no SSH, no Step Functions, no lifecycle hook.

**Secret handling:** API credentials never reach instances. The Activation Lambda exchanges the
API token for a short-lived enrollment token in memory, writes it to Secrets Manager, and instances
read only the bootstrap secret. Instance IAM roles have no access to the API credentials secret.

---

## Documentation Index

| Document | Contents |
|---|---|
| [docs/QUICKSTART.md](docs/QUICKSTART.md) | Prerequisites checklist, three-step deploy, console alternative |
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Full parameter reference, preflight checks, deploy options, verification |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | VPC design, traffic flows, IAM roles, HA, cost estimate |
| [docs/OPERATIONS.md](docs/OPERATIONS.md) | Scaling, monitoring, log groups, AMI upgrade procedure |
| [docs/SECURITY.md](docs/SECURITY.md) | IAM least privilege, secrets handling, encryption |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Issue/Cause/Solution format, diagnostic commands |
| [CLAUDE_DEV.md](CLAUDE_DEV.md) | Template conventions, resource inventory, development rules |

---

## Rules

- **The template (~71 KB) must be deployed via S3 `--template-url`** — it exceeds the
  51 KB direct-upload limit. Run `scripts/deploy-artifacts.sh <region>` to create the S3 bucket.
- **All Lambda functions use inline `ZipFile` code** — no S3 Lambda artifacts are required.
- **AMI defaults are us-west-1 only** — override `GatewayAmiId` and `DlpodAmiId` for other regions.
- **No ASG has an `UpdatePolicy`** — changing an AMI ID updates the launch template only; run
  `aws autoscaling start-instance-refresh` manually to roll instances.
- **Guardrails instance types must have local NVMe instance storage** — do not add EBS-only types
  to `GuardrailsInstanceType` AllowedValues, and do not remove the NVMe mount / Docker data-root
  logic from the Guardrails UserData. Boot on NVMe was measured much faster than on EBS.
