# Netskope AI Gateway — CloudFormation Reference Architecture

CloudFormation reference architecture for deploying [Netskope AI Gateway](https://docs.netskope.com/en/ai-gateway/) (AIG)
together with [DLP On Demand](https://docs.netskope.com/en/data-loss-prevention-on-demand/) (DLPoD) in a
single stack. A new VPC is created — no pre-existing networking is required. Both services are
configured, enrolled, and wired together before entering service, with no manual steps.

![Architecture](docs/architecture.png)

---

## Template

This repository contains one CloudFormation template, `templates/gateway-combined.yaml`, which
deploys AIG + DLP On Demand (+ optional AI Guardrails) in a single stack. Standalone templates for
deploying AIG or DLPoD individually live in a separate repository:
[AWS-POV-Templates-CFT](https://github.com/jharris-ns/AWS-POV-Templates-CFT).

---

## Start Here

| I want to… | Go to |
|---|---|
| Deploy this quickly — I have my Netskope credentials ready | [QUICKSTART.md](docs/QUICKSTART.md) |
| Understand the VPC design, traffic flows, and HA architecture | [ARCHITECTURE.md](docs/ARCHITECTURE.md) |
| Review IAM roles, secrets handling, and encryption | [SECURITY.md](docs/SECURITY.md) |
| Deploy with full parameter documentation | [DEPLOYMENT.md](docs/DEPLOYMENT.md) |
| Operate a running deployment — scaling, monitoring, upgrades | [OPERATIONS.md](docs/OPERATIONS.md) |
| Diagnose a failing deployment or instance | [TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) |

---

## What This Provides

The AI Gateway sits inline between applications and LLM providers (Bedrock, OpenAI, etc.),
inspecting every prompt and response before it crosses the network boundary. Applications point
at the gateway's HTTPS endpoint and require no code changes — the gateway presents an
OpenAI-compatible API regardless of the upstream model.

**Controls applied to every request and response:**

| Control | What it enforces |
|---|---|
| **Data loss prevention** | Detects and blocks sensitive data in prompts and responses — PII, credentials, regulated content — using Netskope DLP policies. With DLP On Demand, content is scanned locally inside your VPC; no data leaves your AWS account for DLP processing. |
| **Prompt injection detection** | Identifies attempts to override system instructions or exfiltrate data through the model. Detection runs on the gateway using built-in rules; the optional AI Guardrails service adds a locally-hosted ML classifier for higher accuracy. |
| **Access control** | Enforces which applications and users can reach which models, based on Netskope policy. Requests that fail policy are rejected at the gateway before reaching the LLM provider. |
| **Rate limiting** | Caps request volume per application or user to control cost and prevent abuse. |
| **Audit logging** | Records all requests and responses — including blocked ones — to Netskope's management plane for visibility and compliance review. |

**AI Guardrails (optional):** A GPU-backed Auto Scaling Group running Netskope's
`aisecurityllm` container provides ML-based prompt injection and content safety classification.
The model runs entirely within your VPC on NVIDIA GPU instances (g4dn or g5 family — these types
are required because the image and Docker storage are placed on the local NVMe instance store for
fast boot), behind an internal ALB at `guardrails.aigw.internal`. Enable it by setting
`GuardrailsImageS3Bucket` (S3 bucket holding the `aisecurity-llm.tgz` tarball) and `GuardrailsAmiId`;
AIG is wired to it automatically at enrollment.

---

## How It Works

```
Internet → AI Gateway ALB (HTTPS:443)
               ↓
         AI Gateway instances (Auto Scaling Group, private subnets)
               ↓ DLP inspection
         DLP On Demand ALB (internal, dlp.aigw.internal)
               ↓
         DLP On Demand instances (Auto Scaling Group, private subnets)
```

At stack creation, a custom resource generates the DLP On Demand TLS certificate and stores it in
SSM Parameter Store and Secrets Manager; DLP On Demand instances receive it in their `bootstrap.json`
and the AI Gateway ASG is held until the DLP On Demand targets are healthy. Each time an AI Gateway
instance launches, the Activation Lambda registers it with the Netskope tenant and writes the
enrollment token together with the DLP On Demand endpoint and certificate (and the Guardrails host,
if deployed) into the AI Gateway bootstrap secret. The instance reads that secret from AWS Secrets
Manager at boot, self-enrolls, and begins forwarding content to DLP On Demand immediately. No manual
coordination between the two services is required.

---

## AWS Services Used

| Service | Purpose |
|---|---|
| **EC2** | AI Gateway and DLP On Demand instances |
| **Auto Scaling** | Instance lifecycle management; launch/terminate hooks on the AI Gateway ASG only |
| **Elastic Load Balancing** | Internet-facing ALB (AI Gateway) and internal ALB (DLP On Demand) |
| **VPC** | Isolated network: public subnets (ALBs, NAT Gateway), private subnets (instances) |
| **Lambda** | Four inline functions: AI Gateway activation/deregistration, cert generation, DLP On Demand `bootstrap.json` builder, ALB readiness gate |
| **SNS** | Delivers Auto Scaling lifecycle events to Lambda functions |
| **Secrets Manager** | AI Gateway bootstrap secret, Netskope API credentials, DLP On Demand license key, DLP On Demand TLS key |
| **Systems Manager Parameter Store** | DLP On Demand ALB certificate PEM (`/<stack>/dlpod-cert`), auto-generated AI Gateway ALB certificate PEM (`/<stack>/aig-cert`), AI Gateway appliance IDs |
| **ACM** | TLS certificates — auto-generated for both ALBs, or user-provided for the AI Gateway ALB |
| **Route 53** | Private hosted zone (`aigw.internal`) for DLP On Demand and optional Guardrails internal DNS |
| **CloudWatch Logs** | Lambda log groups |
| **IAM** | Instance profiles, Lambda execution roles, lifecycle SNS publishing roles |
| **S3** | Hosts the template for `--template-url` (the template exceeds the 51 KB direct-upload limit) |

---

## IAM Requirements

The deploying IAM principal needs `CAPABILITY_NAMED_IAM` to create the IAM roles the stack
provisions. Required permissions:

| Service | Actions |
|---|---|
| CloudFormation | `cloudformation:*` |
| EC2 | `ec2:*` |
| Elastic Load Balancing | `elasticloadbalancing:*` |
| Auto Scaling | `autoscaling:*` |
| Lambda | `lambda:*` |
| SNS | `sns:*` |
| Secrets Manager | `secretsmanager:*` |
| SSM | `ssm:PutParameter`, `ssm:GetParameter`, `ssm:DeleteParameter`, `ssm:AddTagsToResource` |
| ACM | `acm:*` |
| Route 53 | `route53:*` |
| CloudWatch Logs | `logs:*` |
| IAM | `iam:CreateRole`, `iam:DeleteRole`, `iam:GetRole`, `iam:PutRolePolicy`, `iam:DeleteRolePolicy`, `iam:AttachRolePolicy`, `iam:DetachRolePolicy`, `iam:PassRole`, `iam:TagRole`, `iam:UntagRole`, `iam:CreateInstanceProfile`, `iam:DeleteInstanceProfile`, `iam:GetInstanceProfile`, `iam:AddRoleToInstanceProfile`, `iam:RemoveRoleFromInstanceProfile` |
| S3 (bucket) | `s3:CreateBucket`, `s3:ListBucket`, `s3:GetBucketLocation` on `arn:aws:s3:::netskope-aigw-templates-*` |
| S3 (objects) | `s3:GetObject`, `s3:PutObject` on `arn:aws:s3:::netskope-aigw-templates-*/*` |
| STS | `sts:GetCallerIdentity` |

The ready-to-attach IAM policy JSON and the production-hardening note (scoping the `iam:*`
statement to your stack prefix) are maintained in one place:
[DEPLOYMENT.md — AWS Permissions](docs/DEPLOYMENT.md#aws-permissions).

---

## Security Highlights

- **No secrets on instances** — API credentials never reach EC2 instances. The Activation Lambda
  exchanges the token for a short-lived enrollment token, written to Secrets Manager. Instances
  read only the bootstrap secret.
- **Instances in private subnets** — No public IP addresses on AI Gateway, DLP On Demand, or
  Guardrails instances. Inbound access is exclusively through load balancers.
- **Least-privilege IAM** — Eight dedicated IAM roles (seven, plus `GuardrailsRole` when the
  Guardrails tier is deployed). Each role has only the permissions its specific function requires.
  See [SECURITY.md — IAM Roles and Permissions](docs/SECURITY.md#iam-roles-and-permissions).
- **Sensitive parameters masked** — `NetskopeApiToken` and `DlpodLicenseKey` use `NoEcho: true`
  and are never shown in CloudFormation events or the console.
- **DLP runs inside your VPC** — Content sent to DLP On Demand for inspection never leaves your
  AWS account.
- **All traffic encrypted** — External HTTPS via ACM, AIG to DLPoD HTTPS with ACM-imported cert,
  Lambda to AWS services via TLS SDK.
- **Caution: self-signed cert by default** — The auto-generated AIG ALB certificate causes browser
  warnings. For production deployments, provide an ACM-issued certificate via `AcmCertificateArn`.
  See [DEPLOYMENT.md — ACM Certificate](docs/DEPLOYMENT.md#acm-certificate-optional).

---

## Cost Estimate

Approximate monthly cost in us-west-1 (on-demand pricing, minimum deployment):

| Configuration | Estimated monthly cost |
|---|---|
| Minimum: 1 AIG (`m5.4xlarge`) + 1 DLPoD (`c5a.4xlarge`) | ~$1,070–$1,140 |
| Scaled: 4 AIG + 2 DLPoD (maximum defaults) | ~$3,200–$3,500 |
| + AI Guardrails GPU tier (`g4dn.xlarge` per instance) | +~$380/instance/month |

AWS services (NAT Gateway, ALBs, Lambda, Secrets Manager, Route 53) add ~$80–$140/month.
Reserved Instances or Savings Plans reduce EC2 costs by 30–60% for steady workloads.

See [ARCHITECTURE.md — Cost Estimate](docs/ARCHITECTURE.md#cost-estimate) for a full breakdown.

---

## Documentation

| Document | Audience | Contents |
|---|---|---|
| [docs/QUICKSTART.md](docs/QUICKSTART.md) | Netskope customer / SE | Prerequisites checklist, three-step deploy, console alternative |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | AWS Architect | VPC design, traffic flows, IAM roles, HA, cost estimate, Well-Architected alignment |
| [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md) | Customer / DevOps | Full prerequisites, parameter reference, deploy commands, verification steps |
| [docs/SECURITY.md](docs/SECURITY.md) | InfoSec | IAM least privilege, secrets handling, encryption, CFN security practices |
| [docs/OPERATIONS.md](docs/OPERATIONS.md) | DevOps Engineer | Startup sequence, scaling, monitoring, log groups, AMI upgrade |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | DevOps Engineer | Issue/Cause/Solution format with log patterns |
| [CLAUDE_DEV.md](CLAUDE_DEV.md) | Developer | Template conventions, resource inventory, development rules |

---

## Glossary

| Term | Definition |
|---|---|
| **Enrollment token** | One-time token generated by the Netskope API during appliance registration. Passed to the AI Gateway instance via Secrets Manager at boot. |
| **Bootstrap (DLP On Demand)** | First-boot configuration by `nsbootstrap.service` from a `bootstrap.json` delivered in EC2 UserData — TLS certificate and key, license key, DNS server and persona. No SSH or orchestration is involved. |
| **Bootstrap secret** | AWS Secrets Manager secret read by the AI Gateway at boot. Written by the Activation Lambda at each AI Gateway launch; contains the enrollment token, the DLP On Demand endpoint and certificate, and the Guardrails host if deployed. |
| **Lifecycle hook** | Auto Scaling mechanism that holds an instance in a wait state while automation runs. Used on the AI Gateway ASG only (120 s heartbeat). DLP On Demand and Guardrails have no lifecycle hooks. |
| **Management plane** | Netskope's cloud-hosted control plane. Appliances register with it to receive security policies, configuration updates, and DLP profiles. |
| **Readiness gate** | Two mechanisms that block the AI Gateway ASG from launching at stack creation. The DLP On Demand gate is a custom resource whose Lambda polls the DLP On Demand ALB target group (up to 840 s). The Guardrails gate (if deployed) is an `AWS::CloudFormation::WaitCondition` (60-minute timeout) signalled from the first Guardrails instance's UserData once its local `/ping` health check returns 200; UserData gives up and signals FAILURE after 15 minutes. |

---

## Related Resources

- [Netskope AI Gateway Documentation](https://docs.netskope.com/en/ai-gateway/)
- [Deploy AI Gateway on Netskope Portal](https://docs.netskope.com/en/deploy-ai-gateway-on-netskope-portal/)
- [AI Gateway Sizing Guidelines](https://docs.netskope.com/en/ai-gateway-sizing-guidelines/)
- [DLP On Demand Documentation](https://docs.netskope.com/en/data-loss-prevention-on-demand/)
- [Netskope RBAC V3 Overview](https://docs.netskope.com/en/netskope-rbac-v3-overview/)
