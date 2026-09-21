# Architecture Overview — AI Gateway + DLP On Demand

AWS reference architecture for deploying Netskope AI Gateway (AIG) and DLP On Demand (DLPoD)
together using CloudFormation. This document explains each design decision through the lens of
AWS best practices and the
[AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html).

## Table of Contents

- [Architecture Diagram](#architecture-diagram)
- [Component Overview](#component-overview)
  - [VPC and Subnet Design](#vpc-and-subnet-design)
  - [AI Gateway (AIG) — Internet-Facing Tier](#ai-gateway-aig--internet-facing-tier)
  - [DLP On Demand (DLPoD) — Internal Inspection Tier](#dlp-on-demand-dlpod--internal-inspection-tier)
  - [AI Guardrails (Optional) — GPU Inference Tier](#ai-guardrails-optional--gpu-inference-tier)
  - [Instance Sizing and Throughput](#instance-sizing-and-throughput)
  - [Certificate Management](#certificate-management)
  - [IAM Design](#iam-design)
- [Traffic Flows](#traffic-flows)
- [Security Design](#security-design)
- [High Availability Design](#high-availability-design)
- [Cost Estimate](#cost-estimate)
- [Well-Architected Alignment](#well-architected-alignment)

---

## Architecture Diagram

```
                         Internet
                             │
                   ┌─────────▼─────────┐
                   │  AIG ALB (HTTPS)  │  ← Public subnets AZ1 + AZ2
                   │  Internet-facing  │
                   └─────────┬─────────┘
                             │
               ┌─────────────┼─────────────┐
               │             │             │
       ┌───────▼──────┐      │      ┌──────▼────────┐
       │  AIG Instance│      │      │  AIG Instance │  ← Private subnets
       │     AZ1      │      │      │     AZ2       │
       └───────┬──────┘      │      └──────┬────────┘
               │             │             │
               └─────────────┼─────────────┘
                             │ DLP inspection
                   ┌─────────▼─────────┐
                   │  DLPoD ALB        │  ← Private subnets AZ1 + AZ2
                   │  dlp.aigw.internal│    (Route 53 private zone)
                   └─────────┬─────────┘
                             │
               ┌─────────────┼─────────────┐
       ┌───────▼──────┐             ┌──────▼────────┐
       │ DLPoD Instance│            │ DLPoD Instance│
       │     AZ1       │            │     AZ2       │
       └───────────────┘            └───────────────┘

NAT Gateway (public subnet AZ1) → Netskope API, LLM providers
S3 Gateway Endpoint (both route tables) → S3 (no NAT Gateway cost)

[optional] AIG instances → Guardrails ALB (HTTP:8080, guardrails.aigw.internal) → Guardrails GPU ASG
```

---

## Component Overview

### VPC and Subnet Design

The template creates a new VPC — no pre-existing networking is required. The only network
parameter is `VpcCidr` (default `10.0.0.0/16`); the four subnets are derived from it with
`Fn::Cidr` as consecutive `/24` blocks.

| Subnet tier | AZs | Hosts | CIDR (default) |
|---|---|---|---|
| Public | AZ1 + AZ2 | AIG ALB, NAT Gateway (AZ1 only) | `10.0.0.0/24`, `10.0.1.0/24` |
| Private | AZ1 + AZ2 | AIG instances, DLPoD instances, DLPoD internal ALB, Guardrails ALB + instances (optional) | `10.0.2.0/24`, `10.0.3.0/24` |

No compute resources run in public subnets. The NAT Gateway provides outbound internet access
for private-subnet instances — enrollment, API calls to the Netskope management plane, DLPoD's
management-plane connection, and LLM provider traffic all exit through the NAT Gateway. An S3
Gateway Endpoint is attached to both the public and private route tables so S3 traffic (e.g.
Guardrails image download, template reads) stays on the AWS backbone and does not traverse the
NAT Gateway. The stack's Lambda functions are not VPC-attached and do not use the NAT Gateway.

> *Well-Architected [SEC05-BP02](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/sec_network_protection_create_layers.html):
> Place workloads in private subnets unless they require direct inbound internet access.*

### AI Gateway (AIG) — Internet-Facing Tier

| Attribute | Value |
|---|---|
| Instance type (default) | `m5.4xlarge` (16 vCPU / 64 GB RAM) |
| AMI | Netskope AI Gateway (AWS Marketplace) |
| Deployment | Auto Scaling Group across private AZ1 + AZ2 |
| ALB | Internet-facing, public subnets AZ1 + AZ2, HTTPS port 443 |
| Default capacity | Min 1, Max 4 |
| Health check | ELB (HTTPS `GET /` on 443, 200–499 accepted, every 10 s), 600 s grace period |
| Auto-scale trigger | Average CPU > `ScaleOutCpuThreshold` (default 70 %) for two consecutive 5-minute periods (`GreaterThanThreshold`) — scale-out only |
| Enrollment | Reads bootstrap secret from Secrets Manager at boot; self-enrolls autonomously |

The AIG presents an OpenAI-compatible HTTPS API to clients. Every request and response passes
through the gateway's inline inspection — DLP, prompt injection detection, access control, rate
limiting, and audit logging — before reaching the upstream LLM provider.

### DLP On Demand (DLPoD) — Internal Inspection Tier

| Attribute | Value |
|---|---|
| Instance type (default) | `c5a.4xlarge` (16 vCPU / 32 GB RAM) — Netskope's "Small" (proof-of-concept) tier; see [Instance Sizing and Throughput](#instance-sizing-and-throughput) |
| AMI | Netskope DLP On Demand (shared to your account from the Netskope console) |
| Root volume | 351 GB gp3, `Encrypted: true` — matches Netskope's minimum disk requirement for the appliance |
| Deployment | Auto Scaling Group across private AZ1 + AZ2 |
| ALB | Internal, private subnets, HTTPS port 443 |
| DNS | `dlp.aigw.internal` (Route 53 private hosted zone) |
| Default capacity | Min 1, Max 4 |
| Health check | ELB (HTTPS `GET /` on 443, 200–499 accepted, every 30 s), 1800 s grace period |
| Bootstrap | `bootstrap.json` delivered via EC2 UserData; applied by the appliance's `nsbootstrap.service` at first boot |

DLPoD receives content from the AIG over HTTPS, applies DLP policies locally inside the VPC, and
returns a verdict. Content never leaves your AWS account for DLP processing.

There is no lifecycle hook, SNS topic, or per-instance Lambda on the DLPoD tier. The launch
template UserData already contains everything the appliance needs (TLS server cert + key, license
key, DNS resolver `169.254.169.253`, `dlp-on-demand` persona), so scale-outs and replacements
need no orchestration.

**Time to ready.** The ALB health check only tests that the appliance answers on port 443, which
typically happens 5–10 minutes after launch. Netskope's
[DLP On Demand setup guide](https://docs.netskope.com/en/dlpondemandconfig) states that the
appliance needs roughly 30 minutes after tethering to fully initialise (download DLP profiles and
report ready). The ALB health check therefore passes before the appliance is fully operational;
confirm DLP-profile readiness in the Netskope console (Security Cloud Platform > On-Premises
Infrastructure) before relying on inspection results.

**Egress requirements.** DLPoD egress goes through the NAT Gateway with an open outbound
security-group rule. If you add egress filtering (a firewall, proxy, or restrictive NACL), the
appliance must still be able to reach, per the Netskope setup guide:

- the Netskope IP ranges for your tenant
- Amazon S3 (`*.s3-us-west-1.amazonaws.com` is listed by Netskope; the stack's S3 Gateway
  Endpoint covers in-region S3 traffic)
- `config-<tenant>.goskope.com`
- `callhome-<tenant>.goskope.com`
- the `dlpappliancegw.*.goskope.com` set

Netskope advises against TLS interception of this management-plane traffic — pass it through
any inspecting proxy unmodified.

> *Well-Architected [SEC09-BP02](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/sec_protect_data_transit.html):
> Enforce encryption in transit and keep sensitive data within your network boundary.*

### AI Guardrails (Optional) — GPU Inference Tier

Deployed only when `GuardrailsImageS3Bucket` is set (the `DeployGuardrails` condition).

| Attribute | Value |
|---|---|
| Instance type (default) | `g4dn.xlarge` (4 vCPU / 16 GB RAM / NVIDIA T4); `AllowedValues` limited to `g4dn.xlarge`, `g4dn.2xlarge`, `g5.xlarge`, `g5.2xlarge` |
| AMI | AWS Deep Learning Base GPU AMI (`GuardrailsAmiId`) — NVIDIA driver preinstalled |
| Root volume | 100 GB gp3, `Encrypted: true` |
| Deployment | Auto Scaling Group across private AZ1 + AZ2, Min 1, Max 4 |
| ALB | Internal, private subnets, HTTP on `GuardrailsContainerPort` (default 8080), `guardrails.aigw.internal` |
| Health check | ALB: HTTP `GET /ping` expecting 200, every 30 s. ASG: `HealthCheckType: EC2` (no ALB-driven replacement, no grace period) |
| Container | `aisecurityllm` image loaded from the `aisecurity-llm.tgz` tarball in S3; `docker run --gpus all`, `--restart=unless-stopped` |
| Readiness gate | `GuardrailsReadinessGate` (`AWS::CloudFormation::WaitCondition`, `Count: 1`, `Timeout: 3600`) |

The AIG reaches this tier via the `ai_guardrails.host` entry
(`http://guardrails.aigw.internal:8080/invocations`) that the Activation Lambda writes into the
AIG bootstrap secret at every AIG launch.

**Why the instance type is restricted to NVMe-equipped types.** Every allowed type ships a local
NVMe instance store. The launch template UserData mounts it (the Deep Learning AMI auto-mounts it
at `/opt/dlami/nvme`; otherwise UserData finds the `NVMe Instance Storage` device with `lsblk`,
formats it ext4, and mounts it), downloads `aisecurity-llm.tgz` onto it, and moves Docker's
`data-root` onto it before `docker load`. Testing showed this boots much faster than running the
multi-GB image load on the EBS root volume. If no instance-store device is present, UserData falls
back to `/tmp` on the 100 GB gp3 root — a much slower path that risks exceeding the 15-minute
container-health budget in UserData. Because the instance store is ephemeral, every launch
re-downloads the tarball from S3 and re-loads the image; S3 bucket region (the bucket must be in
the stack's region) and instance-store presence together determine boot time. Do not widen
`GuardrailsInstanceType` to EBS-only types without re-testing.

**Readiness gate.** Unlike the DLPoD tier, the Guardrails gate does not use the readiness Lambda.
`GatewayAutoScalingGroup` depends on `GuardrailsReadinessGate`, a CloudFormation WaitCondition
that is satisfied by a cfn-signal-style `curl` from the Guardrails instance UserData once the
local `/ping` returns 200. UserData polls 90 times at 10-second intervals (15 minutes) and then
signals `FAILURE`, which fails the WaitCondition and rolls the stack back. The WaitCondition's own
timeout is 3600 s; with `Count: 1` the first successful signal releases the AIG ASG. Nothing is
written to `/aws/lambda/<stack>-dlpod-readiness` for this gate — look at
`describe-stack-events` and `/var/log/user-data.log` on the instance instead.

The Guardrails tier is the only tier reachable for diagnostics (SSM Session Manager via
`AmazonSSMManagedInstanceCore`); AIG and DLPoD instances have no interactive access path.

### Instance Sizing and Throughput

| Service | Instance type | vCPU | Memory | Instance store | Notes |
|---|---|---|---|---|---|
| AI Gateway | `m5.4xlarge` (default) | 16 | 64 GB | — | Standard DLP + guardrails (CPU-based) |
| AI Gateway | `m6i.4xlarge` | 16 | 64 GB | — | Alternative; newer generation |
| AI Gateway | `c5.4xlarge` | 16 | 32 GB | — | Compute-optimized; lower memory |
| AI Guardrails (optional) | `g4dn.xlarge` (default) | 4 | 16 GB | 125 GB NVMe | NVIDIA T4; `aisecurityllm` container tier |
| AI Guardrails (optional) | `g4dn.2xlarge` | 8 | 32 GB | 225 GB NVMe | NVIDIA T4 |
| AI Guardrails (optional) | `g5.xlarge` | 4 | 16 GB | 250 GB NVMe | NVIDIA A10G |
| AI Guardrails (optional) | `g5.2xlarge` | 8 | 32 GB | 450 GB NVMe | NVIDIA A10G |
| DLP On Demand | `c5a.4xlarge` (default) | 16 | 32 GB | — | Netskope "Small" tier — proof-of-concept only |
| DLP On Demand | `c5ad.4xlarge` | 16 | 32 GB | 2 × 300 GB NVMe | Netskope "Small" tier — proof-of-concept only |
| DLP On Demand | `c5a.8xlarge` | 32 | 64 GB | — | Netskope "Medium" tier (production) — use where `c5ad` is unavailable |
| DLP On Demand | `c5ad.8xlarge` | 32 | 64 GB | 2 × 600 GB NVMe | Netskope "Medium" tier (production, recommended) |
| DLP On Demand | `c5a.16xlarge` | 64 | 128 GB | — | Netskope "Large" tier (production) — use where `c5ad` is unavailable |
| DLP On Demand | `c5ad.16xlarge` | 64 | 128 GB | 2 × 1200 GB NVMe | Netskope "Large" tier (production, recommended) |

**Instance store:** Guardrails types are restricted to those with local NVMe because the image
tarball and Docker storage are placed there (see the
[Guardrails section](#ai-guardrails-optional--gpu-inference-tier)). For DLPoD, the `c5ad`
family is what Netskope lists; the `c5a` equivalents are the documented fallback where `c5ad` is
not offered in a region. All six DLPoD types are in the template's `AllowedValues`.

**DLPoD tiers:** per the
[Netskope DLP On Demand setup guide](https://docs.netskope.com/en/dlpondemandconfig), the
`4xlarge` size is intended only for proof-of-concept testing; Netskope recommends Medium
(`8xlarge`) or Large (`16xlarge`) for production. The template default `c5a.4xlarge` is therefore
a POC default — set `DlpodInstanceType` for production. The 351 GB gp3 root volume in the template
matches Netskope's minimum disk requirement for every tier.

See [AI Gateway Sizing Guidelines](https://docs.netskope.com/en/ai-gateway-sizing-guidelines/)
for request throughput guidance per AIG instance type.

### Certificate Management

Both ALBs use TLS certificates managed by the stack:

| Certificate | ALB | Source | Storage |
|---|---|---|---|
| AIG ALB cert | Internet-facing | Auto-generated self-signed (default), or user-provided ACM ARN | ACM (imported by stack when auto-generated) + SSM `/<stack>/aig-cert` (PEM for clients) |
| DLPoD ALB cert | Internal | Always auto-generated: stack CA + leaf for `dlp.aigw.internal` (365-day validity) | ACM (leaf, imported) + SSM `/<stack>/dlpod-cert` (CA PEM) + Secrets Manager `<stack>-dlpod-cert-key` (CA + leaf + private key) |

The `CertGeneratorFunction` custom resource (`<stack>-certgen`, inline Lambda) runs at stack
creation before any instances launch. For DLPoD it generates a two-tier hierarchy, imports the leaf
cert to ACM for the DLPoD ALB listener, writes the CA PEM to SSM, and writes the full CA + leaf +
key to `<stack>-dlpod-cert-key`. Two consumers pick this up:

- `DlpodBootstrapBuilderFunction` (`<stack>-dlpod-bootstrap-builder`) reads the cert-key secret and
  the license key and assembles the DLPoD `bootstrap.json` UserData — so every DLPoD instance
  serves the same leaf cert on 443.
- The Activation Lambda reads the CA PEM from `/<stack>/dlpod-cert` at every AIG launch and
  writes it into the bootstrap secret as `dlp.certificate` — so the first AIG instance that starts
  already trusts the DLPoD endpoint.

The same function generates the AIG ALB cert when `AcmCertificateArn` is left empty.

> **Note:** The self-signed AIG ALB certificate causes browser security warnings and requires
> clients to trust the cert or disable TLS verification. For production deployments where clients
> cannot be configured to skip certificate verification, provide an ACM-issued certificate ARN
> via `AcmCertificateArn`.

### IAM Design

Seven IAM roles (eight with the optional Guardrails tier) enforce least privilege: each Lambda
function, each instance tier, and the Auto Scaling SNS publisher has its own role, scoped to the
specific secret, parameter, log group, and ASG ARNs it needs. The only `Resource: "*"` grants are
on actions that cannot be ARN-scoped (`ec2:DescribeInstances`,
`elasticloadbalancing:DescribeTargetHealth`, ACM import). AIG instances never hold Netskope API
credentials — the Activation Lambda exchanges the API token for an enrollment token in memory and
writes only that to the bootstrap secret — and DLPoD instances have no Secrets Manager or SSM
access at all.

The complete role inventory and per-role permission detail is maintained in
[SECURITY.md — IAM Roles and Permissions](SECURITY.md#iam-roles-and-permissions).

> *Well-Architected [SEC03-BP01](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/sec_permissions_define.html):
> Define access requirements and enforce least privilege.*

---

## Traffic Flows

### 1. Inbound client traffic (runtime)

```
1. Client sends HTTPS request to AIG ALB DNS name
2. AIG ALB (public subnets) → AIG instance (private subnet)
3. AIG instance inspects request:
   - Access control, rate limiting, prompt injection detection
4. AIG → dlp.aigw.internal (Route 53) → DLPoD internal ALB → DLPoD instance
   - DLPoD applies DLP policies; returns allow/block verdict
5. AIG forwards allowed requests to upstream LLM provider (via NAT Gateway)
6. LLM response returns through AIG → DLP inspection → client
```

### 2. AIG enrollment (stack creation and every scale-out, per instance)

```
1. ASG launches AIG instance → lifecycle hook holds it in Pending:Wait (120s)
2. SNS delivers lifecycle event → Activation Lambda (<stack>-aig-activation)
3. Activation Lambda:
   a. Reads API token from <stack>-netskope-credentials (Secrets Manager)
   b. Calls Netskope REST API → registers appliance → receives enrollment token
   c. Writes appliance ID to SSM /aig/<stack>/<instance-id>
   d. Reads the DLPoD CA cert PEM from SSM /<stack>/dlpod-cert
   e. Writes <stack>-aig-bootstrap (Secrets Manager): enrollment token + dlp {host, certificate}
      (+ ai_guardrails {host} when the Guardrails tier is deployed)
   f. Calls CompleteLifecycleAction: CONTINUE (any exception → ABANDON, instance replaced)
4. AIG instance reads bootstrap secret at boot → self-enrolls with Netskope tenant
5. AIG configures DLP forwarding to https://dlp.aigw.internal
```

### 3. DLPoD bootstrap (stack creation and every scale-out, per instance)

Two custom resources run once, before the DLPoD ASG exists:

```
Stack creation (pre-launch)
1. DlpodAlbCertificate → CertGeneratorFunction (<stack>-certgen):
   generates CA + leaf for dlp.aigw.internal, imports leaf to ACM, writes CA PEM to
   SSM /<stack>/dlpod-cert, writes CA + leaf + key to <stack>-dlpod-cert-key
2. DlpodBootstrapPart1 / Part2 → DlpodBootstrapBuilderFunction (<stack>-dlpod-bootstrap-builder):
   reads <stack>-dlpod-cert-key and <stack>-dlpod-credentials (license key), assembles
   bootstrap.json {dlpaas: cert/key/CA chain, dns: 169.254.169.253, system: licensekey,
   persona: dlp-on-demand}, base64-encodes it in two halves (4 KB custom-resource limit)
   → joined as DlpodLaunchTemplate UserData
```

Then, for each DLPoD instance — no lifecycle hook, SNS, or Lambda involved:

```
Instance launch
1. ASG launches DLPoD instance from DlpodLaunchTemplate → InService immediately
   (HealthCheckType: ELB, HealthCheckGracePeriod: 1800s)
2. nsbootstrap.service reads bootstrap.json from EC2 UserData at first boot:
   installs TLS server cert + key + CA chain, sets DNS resolver, applies license key,
   sets the dlp-on-demand persona; appliance connects outbound to the Netskope
   management plane via the NAT Gateway
3. DLPoD listens on HTTPS:443 → ALB health check (GET /, 200–499, 2 passes at 30 s) → healthy
   (typically 5–10 minutes after launch; full DLP-profile initialisation takes longer —
   see the DLPoD tier section)
```

**Readiness gates (stack creation only):** `GatewayAutoScalingGroup` depends on
`DlpodReadinessGate` — a custom resource whose inline Lambda polls the DLPoD target group until
every target is healthy (840 s limit) — and, when Guardrails is deployed, on
`GuardrailsReadinessGate`, a CloudFormation WaitCondition signalled from the Guardrails instance
UserData. Both are described in full in
[OPERATIONS.md — Startup Sequence](OPERATIONS.md#startup-sequence).

### 4. DLP inspection (runtime, per request)

```
1. AIG instance resolves dlp.aigw.internal via Route 53 private zone
2. AIG → DLPoD internal ALB (HTTPS:443, stack-generated leaf cert; AIG trusts the stack CA)
3. DLPoD ALB → DLPoD instance (HTTPS:443, same leaf cert served by the appliance)
4. DLPoD scans content; returns verdict
5. AIG applies verdict (allow / block / redact)
```

### 5. AIG scale-out (runtime, automatic)

```
1. CloudWatch alarm: AIG ASG average CPU > 70% (GreaterThanThreshold) for two 5-minute periods
2. Step scaling policy adds one AIG instance
3. New instance → Pending:Wait → Activation Lambda (same flow as #2)
4. New instance: InService → ALB healthy → serving requests
   (enrollment typically completes in 5–15 minutes)
```

There is no scale-in policy. Reducing capacity is a manual operation — see
[OPERATIONS.md — Scaling](OPERATIONS.md#scaling).

---

## Security Design

### Network Isolation

| Resource | Ingress allowed from | Egress allowed to |
|---|---|---|
| AIG ALB security group (`<stack>-aig-alb-sg`) | `0.0.0.0/0` port 443 (internet clients) | VPC CIDR port 443 (AIG instances) |
| AIG instance security group (`<stack>-aig-gw-sg`) | AIG ALB SG port 443 | All (NAT Gateway → internet, DLPoD ALB, Guardrails ALB) |
| DLPoD ALB security group (`<stack>-dlpod-alb-sg`) | AIG instance SG port 443 | VPC CIDR port 443 (DLPoD instances) |
| DLPoD instance security group (`<stack>-dlpod-sg`) | DLPoD ALB SG port 443 only — no SSH ingress | All (NAT Gateway → Netskope management plane) |
| Guardrails ALB security group *(optional)* | AIG instance SG port `GuardrailsContainerPort` (8080) | VPC CIDR port 8080 |
| Guardrails instance security group *(optional)* | Guardrails ALB SG port 8080 | All (NAT Gateway → S3) |

AIG and DLPoD instances have no public IP addresses. The only path from the internet to AIG
instances is through the internet-facing ALB. DLPoD instances are reachable only from the
DLPoD ALB on port 443 — no Lambda, bastion, or SSH path exists, and none of the stack's Lambda
functions is VPC-attached. All launch templates require IMDSv2 and set `Encrypted: true` on the
root volume.

### Secrets and Credentials

The design separates the long-lived Netskope API token from the material that instances actually
consume. The API token lives in `<stack>-netskope-credentials`, readable only by the Activation
Lambda. At every AIG launch the Lambda exchanges it for a one-time enrollment token and writes
that — together with the DLPoD CA certificate from SSM and the DLP / Guardrails host names — into
the separate `<stack>-aig-bootstrap` secret, which is the only secret the AIG instance role can
read. Compromise of the bootstrap secret does not expose the API token.

DLPoD instances never read Secrets Manager or SSM. Their TLS key and license key are assembled
once, at stack create/update, by the bootstrap builder Lambda (which reads `<stack>-dlpod-cert-key`
and `<stack>-dlpod-credentials`) and embedded in the launch template UserData (`bootstrap.json`).
That UserData is readable by principals with `ec2:DescribeLaunchTemplateVersions` — see
[SECURITY.md — Known Limitations](SECURITY.md#known-limitations-and-accepted-risks).

The full inventory of the four secrets and three SSM parameter paths — contents, writer, reader,
lifecycle — is maintained in
[SECURITY.md — What's Stored and Where](SECURITY.md#whats-stored-and-where).

---

## High Availability Design

### Multi-AZ Architecture

Both the AIG ASG and DLPoD ASG span AZ1 and AZ2. Both ALBs are deployed across both AZs.
An AZ failure reduces capacity but does not interrupt service — the healthy AZ continues serving
all traffic.

```
AZ1                              AZ2
┌────────────────────────────┐   ┌────────────────────────────┐
│ Public subnet              │   │ Public subnet              │
│   AIG ALB node             │   │   AIG ALB node             │
│   NAT Gateway              │   │                            │
│                            │   │                            │
│ Private subnet             │   │ Private subnet             │
│   AIG Instance(s)          │   │   AIG Instance(s)          │
│   DLPoD Instance(s)        │   │   DLPoD Instance(s)        │
│   DLPoD ALB node           │   │   DLPoD ALB node           │
└────────────────────────────┘   └────────────────────────────┘
```

### Failure Scenarios

| Scenario | Impact | Recovery |
|---|---|---|
| Single AIG instance failure | Reduced capacity; remaining instances continue | ASG replaces automatically; new instance re-enrolls (~5–15 min) |
| Single DLPoD instance failure | Reduced DLP capacity; ALB routes to healthy instances | ASG replaces; new instance self-configures from UserData via `nsbootstrap` (~5–10 min to ALB-healthy) |
| Single Guardrails instance failure *(optional tier)* | AIG guardrails calls fail until a healthy target exists | ASG replaces only on EC2 status-check failure (`HealthCheckType: EC2`); a running instance whose container is unhealthy is **not** replaced automatically — see [OPERATIONS.md — AI Guardrails](OPERATIONS.md#ai-guardrails-only-when-guardrailsimages3bucket-was-set) |
| AZ failure (AIG) | Reduced capacity; other AZ continues | No action required; ASG may launch replacement in healthy AZ |
| AZ failure (DLPoD) | Reduced DLP capacity | Same as above |
| NAT Gateway failure | Instances lose outbound internet; DLP traffic unaffected (internal) | AWS SLA 99.99%; auto-recovers |
| AIG enrollment failure | Instance ABANDONED by the launch hook; replacement launches automatically | Check Activation Lambda logs; see [TROUBLESHOOTING.md](TROUBLESHOOTING.md) |
| DLPoD bootstrap failure | Target never healthy; ASG marks the instance unhealthy after the 30-min grace period and replaces it (no lifecycle hook, so no ABANDON state) | Check DLPoD target health and `/aws/lambda/<stack>-dlpod-bootstrap-builder`; see [TROUBLESHOOTING.md](TROUBLESHOOTING.md#dlp-on-demand-issues) |
| DLPoD not healthy within 840 s at stack creation | `DlpodReadinessGate` fails; stack rolls back before any AIG instance launches | Re-create with `--disable-rollback` to inspect; see [TROUBLESHOOTING.md](TROUBLESHOOTING.md) |
| Guardrails container not healthy within 15 min at stack creation *(optional tier)* | UserData signals `FAILURE` to `GuardrailsReadinessGate`; stack rolls back | Re-create with `--disable-rollback`; read `/var/log/user-data.log` via SSM; see [TROUBLESHOOTING.md](TROUBLESHOOTING.md#ai-guardrails-issues) |

**RPO:** Zero — both services are stateless. Configuration is stored in CloudFormation and
Netskope's management plane.

**RTO per component:**

| Scope | RTO |
|---|---|
| Single AIG instance | 5–15 minutes (auto-replaced and re-enrolled) |
| Single DLPoD instance | 5–10 minutes to ALB-healthy (auto-replaced; `nsbootstrap` applies UserData at first boot) |
| AZ failure | 0 seconds (healthy AZ continues immediately) |
| Full stack recreate | 12–18 minutes to `CREATE_COMPLETE` (DLPoD ~5–10 min, then AIG ~5–15 min); longer with Guardrails |

---

## Cost Estimate

Approximate monthly cost for the default deployment in us-west-1 (1 AIG + 1 DLPoD instance),
on-demand pricing. Costs vary by region and actual traffic volume.

| Resource | Quantity | Estimated monthly cost |
|---|---|---|
| EC2 — AIG (`m5.4xlarge`) | 1 instance, on-demand | ~$550 |
| EC2 — DLPoD (`c5a.4xlarge`) | 1 instance, on-demand | ~$445 |
| NAT Gateway (data processing + hours) | 1 NAT GW | ~$35–65 |
| ALB — AIG (internet-facing) | 1 ALB | ~$20–40 |
| ALB — DLPoD (internal) | 1 ALB | ~$18–30 |
| Secrets Manager | 4 secrets | ~$1.60 |
| SSM Parameter Store | Standard parameters | Free |
| Route 53 private hosted zone | 1 zone | ~$0.50 |
| Lambda (4 inline functions) | Custom resources at create/update + AIG lifecycle events | <$1 |
| CloudWatch Logs | Lambda + instance logs | ~$1–5 |
| ACM certificates | 2 certs (imported) | Free |
| S3 Gateway Endpoint | 1 endpoint | Free |
| **Total (default, on-demand)** | | **~$1,070–$1,140/month** |

**Scaling impact:**
- Each additional AIG instance (`m5.4xlarge`): +~$550/month
- Each additional DLPoD instance (`c5a.4xlarge`): +~$445/month
- Moving DLPoD to a production tier (`c5ad.8xlarge` / `c5ad.16xlarge`) roughly doubles / quadruples
  the DLPoD EC2 line
- Example higher-capacity deployment (4 AIG + 2 DLPoD): ~$3,200–$3,500/month

**AI Guardrails (optional, `GuardrailsImageS3Bucket` set):**
- `g4dn.xlarge` per GPU instance: +~$380/month
- `g5.xlarge` per GPU instance: +~$760/month
- Guardrails internal ALB: +~$18–30/month
- S3 storage for the `aisecurity-llm.tgz` tarball: ~$0.023/GB-month; each Guardrails launch
  re-downloads it (in-region via the S3 Gateway Endpoint, so no NAT data-processing charge)

> These are estimates for planning purposes. Use [AWS Pricing Calculator](https://calculator.aws/)
> with your actual region, instance counts, and expected traffic volumes for a precise figure.
> Reserved Instances or Savings Plans reduce EC2 costs by 30–60% for steady-state workloads.

---

## Well-Architected Alignment

| Pillar | Design decision | Implementation |
|---|---|---|
| **Security** | Least-privilege IAM — 7 dedicated roles (8 with Guardrails), no shared credentials | Each role scoped to its specific function; AIG instances never hold API credentials; DLPoD instances hold no secrets access at all |
| **Security** | API credentials never in user data or environment variables | API token → Secrets Manager → Lambda (memory only) → bootstrap secret → AIG instance reads at boot |
| **Security** | No public IP on compute instances; no SSH | AIG and DLPoD instances in private subnets; inbound only through ALBs; no security group opens port 22 |
| **Security** | Encryption in transit and at rest | All external traffic TLS; AIG→DLPoD HTTPS with a stack-generated CA/leaf; `Encrypted: true` on all EBS root volumes; IMDSv2 required |
| **Security** | Sensitive parameters masked | `NetskopeApiToken` and `DlpodLicenseKey` are `NoEcho: true` |
| **Reliability** | Multi-AZ deployment | All ASGs and ALBs span AZ1 + AZ2 |
| **Reliability** | Auto-replacement on failure | ELB health checks on the AIG and DLPoD ASGs (AIG also has a launch lifecycle hook); the Guardrails ASG uses EC2 status checks only |
| **Reliability** | Startup ordering enforced | `CertGeneratorFunction` and bootstrap builder run before instances launch; `DlpodReadinessGate` (and `GuardrailsReadinessGate` WaitCondition) block the AIG ASG until the dependent tiers are serving |
| **Reliability** | No manual enrollment steps | Activation Lambda handles AIG enrollment; DLPoD self-configures from `bootstrap.json` via `nsbootstrap.service` |
| **Operational Excellence** | Infrastructure as code | Single CloudFormation template, all Lambda code inline (`ZipFile`); all resources version-controlled |
| **Operational Excellence** | Lifecycle automation | ASG hook → SNS → Lambda handles every AIG lifecycle event; DLPoD needs none — its launch template UserData is complete |
| **Cost Optimization** | Step scaling (scale-out) | AIG scale-out triggered by the CPU alarm; scale-in is a manual operation (no scale-in policy is defined) |
| **Cost Optimization** | S3 Gateway Endpoint | S3 traffic (Guardrails image download, template reads) stays on the AWS backbone — no NAT Gateway data-processing charges |

> *References: [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html),
> [Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html),
> [Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)*
