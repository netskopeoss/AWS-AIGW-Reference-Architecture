# Troubleshooting Guide — AI Gateway + DLP On Demand

Issue/Cause/Solution reference for `templates/gateway-combined.yaml`. Start with the
[Diagnostic Commands](#diagnostic-commands) section to gather state, then jump to the
specific issue.

Terminology: AI Gateway (AIG), DLP On Demand (DLPoD), AI Guardrails, and the Activation Lambda
(`<stack>-aig-activation`).

## Table of Contents

- [Diagnostic Commands](#diagnostic-commands)
- [AWS Access Issues](#aws-access-issues)
  - [Issue: `aws` commands fail with `ExpiredToken`, `AccessDenied`, or `Unable to locate credentials`](#issue-aws-commands-fail-with-expiredtoken-accessdenied-or-unable-to-locate-credentials)
- [AI Gateway Issues](#ai-gateway-issues)
  - [Issue: AIG instance stuck in `Pending:Wait`](#issue-aig-instance-stuck-in-pendingwait)
  - [Issue: AIG instance ABANDONED after 2 minutes](#issue-aig-instance-abandoned-after-2-minutes)
  - [Issue: AIG ALB target stuck unhealthy](#issue-aig-alb-target-stuck-unhealthy)
  - [Issue: AIG enrolled but DLP inspection not working](#issue-aig-enrolled-but-dlp-inspection-not-working)
  - [Issue: Orphaned appliances in the Netskope tenant (manual cleanup via API)](#issue-orphaned-appliances-in-the-netskope-tenant-manual-cleanup-via-api)
- [DLP On Demand Issues](#dlp-on-demand-issues)
  - [Issue: Stack fails at `DlpodReadinessGate` — `did not become healthy within 840s`](#issue-stack-fails-at-dlpodreadinessgate--did-not-become-healthy-within-840s)
  - [Issue: DLPoD instance never becomes healthy (after stack creation)](#issue-dlpod-instance-never-becomes-healthy-after-stack-creation)
  - [Issue: DLPoD ALB target healthy but AIG reports DLP errors](#issue-dlpod-alb-target-healthy-but-aig-reports-dlp-errors)
- [Certificate Issues](#certificate-issues)
  - [Issue: DLPoD cert missing from SSM](#issue-dlpod-cert-missing-from-ssm)
  - [Issue: AIG cannot verify DLPoD TLS certificate](#issue-aig-cannot-verify-dlpod-tls-certificate)
- [AI Guardrails Issues](#ai-guardrails-issues)
  - [Issue: Stack fails at `GuardrailsReadinessGate` — container not healthy](#issue-stack-fails-at-guardrailsreadinessgate--container-not-healthy)
  - [Issue: Create fails immediately with `GuardrailsAmiId is required when GuardrailsImageS3Bucket is set`](#issue-create-fails-immediately-with-guardrailsamiid-is-required-when-guardrailsimages3bucket-is-set)
  - [Issue: Guardrails ASG launch fails with `VcpuLimitExceeded`](#issue-guardrails-asg-launch-fails-with-vcpulimitexceeded)
  - [Issue: AIG enrolled but Guardrails shows unlinked / inspection not applied](#issue-aig-enrolled-but-guardrails-shows-unlinked--inspection-not-applied)
- [Stack Issues](#stack-issues)
  - [Issue: Stack stuck at `CREATE_IN_PROGRESS` for more than 30 minutes](#issue-stack-stuck-at-create_in_progress-for-more-than-30-minutes)
  - [Issue: Stack stuck in `ROLLBACK_FAILED` or `DELETE_FAILED`](#issue-stack-stuck-in-rollback_failed-or-delete_failed)
  - [Issue: Stack deletion hangs](#issue-stack-deletion-hangs)
- [Log Patterns Reference](#log-patterns-reference)

---

## Diagnostic Commands

Run these to understand the current state before diagnosing a specific issue.

```bash
STACK=<stack-name>
REGION=<region>

# Stack status
aws cloudformation describe-stacks --stack-name $STACK \
  --query 'Stacks[0].StackStatus' --output text --region $REGION

# All stack outputs at once
aws cloudformation describe-stacks --stack-name $STACK \
  --query "Stacks[0].Outputs[*].[OutputKey,OutputValue]" \
  --output table --region $REGION

# AIG ASG instance states
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-aig-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION

# DLPoD ASG instance states
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-dlpod-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus]" \
  --output table --region $REGION

# Target group ARNs (reused below)
AIG_TG=$(aws elbv2 describe-target-groups --names $STACK-aig-tg \
  --query "TargetGroups[0].TargetGroupArn" --output text --region $REGION)
DLPOD_TG=$(aws elbv2 describe-target-groups --names $STACK-dlpod-tg \
  --query "TargetGroups[0].TargetGroupArn" --output text --region $REGION)

# DLPoD ALB target health — the only externally visible signal of DLPoD bootstrap progress
aws elbv2 describe-target-health --target-group-arn $DLPOD_TG \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Reason,TargetHealth.Description]" \
  --output table --region $REGION

# Failed / rolled-back resources with reasons
aws cloudformation describe-stack-events --stack-name $STACK --region $REGION \
  --query "StackEvents[?contains(ResourceStatus,'FAILED')].[Timestamp,LogicalResourceId,ResourceStatusReason]" \
  --output table

# AIG bootstrap secret contents
aws secretsmanager get-secret-value \
  --secret-id $STACK-aig-bootstrap \
  --query SecretString --output text --region $REGION

# AIG activation Lambda logs (last 15 min)
aws logs tail /aws/lambda/$STACK-aig-activation --since 15m --region $REGION

# Stack-creation Lambdas (only write logs during create/update)
aws logs tail /aws/lambda/$STACK-certgen --since 1h --region $REGION
aws logs tail /aws/lambda/$STACK-dlpod-bootstrap-builder --since 1h --region $REGION
aws logs tail /aws/lambda/$STACK-dlpod-readiness --since 1h --region $REGION
```

---

## AWS Access Issues

### Issue: `aws` commands fail with `ExpiredToken`, `AccessDenied`, or `Unable to locate credentials`

**Cause:** Every command in this guide runs with whatever AWS credentials are active in your shell.
Missing, expired, or wrong-account credentials fail before any stack diagnosis can start.

**Diagnosis:** `aws sts get-caller-identity` — fails fast if credentials are missing or expired,
otherwise shows which account and role you are acting as.

**Solution:** pick one credential method and set `AWS_PROFILE` before running the commands in this guide.

```bash
# IAM Identity Center / SSO (recommended — short-lived, auto-rotated credentials)
aws configure sso                    # one-time: SSO start URL, region, account, role
aws sso login --profile my-sso-profile
export AWS_PROFILE=my-sso-profile    # re-run `aws sso login` when the session expires (typically 1–12 h)

# Named profile ([my-profile] in ~/.aws/credentials with aws_access_key_id / aws_secret_access_key / region)
export AWS_PROFILE=my-profile

# Assumed role for cross-account deployment ([profile deploy] in ~/.aws/config with
# role_arn = arn:aws:iam::123456789012:role/CloudFormationDeployRole, source_profile = my-profile, region = us-west-1)
export AWS_PROFILE=deploy
```

Avoid long-lived access keys exported as environment variables (they persist in shell history and
process listings) and never deploy with root-account credentials. The deploying principal needs the
permissions listed in [docs/DEPLOYMENT.md](DEPLOYMENT.md) and `CAPABILITY_NAMED_IAM`. References:
[SSO](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html),
[named profiles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html),
[assumed roles](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html).

---

## AI Gateway Issues

### Issue: AIG instance stuck in `Pending:Wait`

**Cause:** The AIG lifecycle hook heartbeat is 120 seconds. The Activation Lambda must register
the appliance with the Netskope API and complete the lifecycle action within that window.

**Diagnosis:**
```bash
aws logs tail /aws/lambda/$STACK-aig-activation --since 10m --region $REGION
```

**Common causes and solutions:**

| Log pattern | Cause | Solution |
|---|---|---|
| `401 Unauthorized` or `403 Forbidden` | `NetskopeApiToken` is wrong, expired, or lacks AIG Administrator permissions | Verify token in Netskope portal: **Settings → Administration → Administrators & Roles → Administrators** |
| `URLError`, `ConnectionError`, or `TimeoutError` | Lambda cannot reach the Netskope API. The Activation Lambda is not VPC-attached — it egresses directly from the Lambda service, so the VPC's NAT Gateway and route tables are not involved | Verify `NetskopeTenantUrl` resolves and is reachable from outside (`curl -sf -o /dev/null -w '%{http_code}\n' -H "Netskope-Api-Token: $NETSKOPE_API_TOKEN" https://<tenant>.goskope.com/api/v2/aig/appliances` should return `200`); confirm the tenant is not IP-allowlisting API access; the Lambda's per-request timeout is 30 s |
| `ParameterNotFound` on `/<stack>/dlpod-cert`, or the `dlp.certificate` value is the literal `pending` | Cert generator custom resource did not overwrite the SSM placeholder | See [Certificate Issues — DLPoD cert missing from SSM](#issue-dlpod-cert-missing-from-ssm) |
| `ResourceNotFoundException` on bootstrap secret | Bootstrap secret not created | Check CloudFormation events for failure on `AigBootstrapSecret` resource |
| `KeyError: 'id'` or `'enrollment_token'` | Netskope API returned an unexpected body (tenant URL wrong, or points at a non-AIG tenant) | Verify `NetskopeTenantUrl` is `https://<tenant>.goskope.com` with no path |

The Lambda calls `CompleteLifecycleAction: ABANDON` on any exception, so a failed launch is
ABANDONED as soon as the error occurs (or after 2 minutes if the Lambda never ran) and a
replacement launches automatically. Check that the underlying issue is resolved before the
replacement arrives.

The full trace of each launch is a single `print` block; search for `Registered appliance` to
confirm success or a Python traceback to see the failure.

---

### Issue: AIG instance ABANDONED after 2 minutes

**Cause:** The Activation Lambda failed or timed out during the 120-second lifecycle hook window.

**Diagnosis:**
```bash
# Check Lambda errors
aws logs tail /aws/lambda/$STACK-aig-activation --since 30m --region $REGION

# Check CloudFormation events for error context
aws cloudformation describe-stack-events --stack-name $STACK --region $REGION \
  --query "StackEvents[?ResourceStatus=='CREATE_FAILED'].[LogicalResourceId,ResourceStatusReason]" \
  --output table
```

**Solution:** The ASG automatically launches a replacement. If replacements are also being
ABANDONED (systematic failure), resolve the root cause — typically a bad API token, network
connectivity, or missing SSM parameter — before further replacements launch.

---

### Issue: AIG ALB target stuck unhealthy

**Cause:** The instance is `InService` in the ASG but the ALB health check (HTTPS GET / on port 443) fails.

**Diagnosis:**
```bash
aws elbv2 describe-target-health --target-group-arn $AIG_TG \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Description]" \
  --output table --region $REGION
```

**Common causes:**

| Health check state | Likely cause | Solution |
|---|---|---|
| `initial` | Instance just launched; enrollment not yet complete | Wait 5–15 minutes from instance launch (ASG grace period is 10 min) |
| `unhealthy` - connection refused | AIG service not running or enrollment incomplete | Check AIG activation Lambda logs; enrollment may have failed |
| `unhealthy` - timeout | Security group not allowing AIG ALB SG → AIG instance port 443 | Check `AigGatewaySecurityGroup` ingress (source `AigAlbSecurityGroup`, port 443) |
| `unhealthy` repeatedly after enrollment succeeded | AIG could not validate the DLPoD (or Guardrails) endpoint it was given in the bootstrap secret | Confirm DLPoD targets are healthy and `dlp.host` is `https://dlp.aigw.internal`; see [AIG enrolled but DLP inspection not working](#issue-aig-enrolled-but-dlp-inspection-not-working) |

---

### Issue: AIG enrolled but DLP inspection not working

**Cause:** DLP On Demand ALB has no healthy targets, the AIG bootstrap secret is missing the DLP
block, or the security group cross-reference is broken.

**Diagnosis — step by step:**

**Step 1: Check DLPoD ALB has healthy targets**
```bash
aws elbv2 describe-target-health --target-group-arn $DLPOD_TG \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State]" \
  --output table --region $REGION
```

If there are no healthy DLPoD targets, DLP inspection fails. At stack creation this cannot happen
(the readiness gate blocks AIG until DLPoD is healthy), so a stack that reached `CREATE_COMPLETE`
and later lost DLPoD health has a replaced or failed DLPoD instance. See
[DLP On Demand Issues](#dlp-on-demand-issues).

**Step 2: Verify AIG bootstrap secret contains the DLP block**
```bash
aws secretsmanager get-secret-value \
  --secret-id $STACK-aig-bootstrap \
  --query SecretString --output text --region $REGION | \
  python3 -c "import json,sys; d=json.load(sys.stdin); print(json.dumps(d.get('dlp',{}), indent=2))"
```

The output should contain `certificate` (a PEM beginning `-----BEGIN CERTIFICATE-----`) and
`host` (`https://dlp.aigw.internal`). The activation Lambda writes this block at every AIG launch
from `/<stack>/dlpod-cert`. If the block is missing, the secret is still the initial template
value — no AIG instance has been through the activation Lambda yet. If `certificate` is the
literal `pending`, the cert generator did not overwrite the SSM placeholder; see
[Certificate Issues](#certificate-issues).

**Step 3: Verify security group allows AIG → DLPoD ALB**
```bash
# Get the DLPoD ALB security group ID from CloudFormation resources
aws cloudformation list-stack-resources --stack-name $STACK --region $REGION \
  --query "StackResourceSummaries[?LogicalResourceId=='DlpodAlbSecurityGroup'].PhysicalResourceId" \
  --output text

# Check its ingress rules
aws ec2 describe-security-group-rules \
  --filters Name=group-id,Values=<dlpod-alb-sg-id> \
  --query "SecurityGroupRules[?!IsEgress].[IpProtocol,FromPort,ToPort,ReferencedGroupInfo.GroupId]" \
  --output table --region $REGION
```

The AIG instance SG should appear as a source for port 443.

---

### Issue: Orphaned appliances in the Netskope tenant (manual cleanup via API)

**Cause:** The Activation Lambda deregisters each appliance during the AIG termination lifecycle hook.
If that run failed (API error, SNS subscription already deleted, stack deleted with instances still
`Terminating:Wait`), the appliance `<stack>-gw-<instance-id>` stays registered in the tenant and its
`/aig/<stack>/<instance-id>` SSM parameter lingers.

**Diagnosis:**
```bash
# Appliance IDs the stack still tracks
aws ssm get-parameters-by-path --path /aig/$STACK/ --query "Parameters[*].[Name,Value]" --output table --region $REGION

# Everything registered in the tenant (compare names against running AIG instances)
curl -s "https://<tenant>.goskope.com/api/v2/aig/appliances" \
  -H "Netskope-Api-Token: $NETSKOPE_API_TOKEN" | python3 -m json.tool
```

**Solution:** deregister the stale appliance and delete its SSM parameter. Use the same RBAC v3
token supplied as `NetskopeApiToken` (exported here as `$NETSKOPE_API_TOKEN` so it never appears in
the command line).

```bash
# Deregister
curl -s -X DELETE "https://<tenant>.goskope.com/api/v2/aig/appliances/<appliance-id>" \
  -H "Netskope-Api-Token: $NETSKOPE_API_TOKEN"

# Remove the stack's record of it
aws ssm delete-parameter --name /aig/$STACK/<instance-id> --region $REGION

# Register manually only if you are re-creating an appliance the Lambda failed to register
# (normally unnecessary — terminating the instance makes the ASG replace it and re-register):
curl -s -X POST "https://<tenant>.goskope.com/api/v2/aig/appliances" \
  -H "Netskope-Api-Token: $NETSKOPE_API_TOKEN" -H "Content-Type: application/json" \
  -d '{"name":"<appliance-name>","host":"<instance-private-ip>","ports":{"https":{"port":443,"enable":true},"http":{"port":80,"enable":false}}}'
```

---

## DLP On Demand Issues

**How DLPoD comes up:** there is no lifecycle hook, Lambda, or orchestration per DLPoD instance.
The launch template UserData carries a base64 `bootstrap.json` (TLS server cert + key + CA chain,
DNS `169.254.169.253`, license key, persona `dlp-on-demand`) assembled once at stack creation by
`DlpodBootstrapBuilderFunction`. The appliance's `nsbootstrap.service` applies it at first boot.
The only externally visible progress signal is the DLPoD ALB target health (HTTPS GET `/` on 443,
any 200–499 response). A DLPoD instance is `InService` in the ASG from launch; the ASG uses ELB
health checks with a 30-minute grace period, after which a still-unhealthy instance is replaced.

### Issue: Stack fails at `DlpodReadinessGate` — `did not become healthy within 840s`

**Cause:** `DlpodReadinessGate` (`Custom::DlpodReadiness`, Lambda `<stack>-dlpod-readiness`) polls
the DLPoD target group every 30 seconds after `DlpodAutoScalingGroup` is created and fails the
stack with `Targets in <stack>-dlpod-tg did not become healthy within 840s` if all targets are not
healthy within 840 s (14 minutes). Typical reasons, in order of likelihood: the DLPoD AMI is slow
to boot on the chosen instance type; `bootstrap.json` was invalid (bad license key, cert-key secret
still the literal `pending`); the appliance cannot reach the Netskope management plane (NAT
Gateway / routing); or the instance could not launch at all (capacity, DLPoD AMI not shared to
this account, or shared for a different region).

**Diagnosis:**
```bash
# What the gate saw
aws logs tail /aws/lambda/$STACK-dlpod-readiness --since 1h --region $REGION
#   "0/1 target(s) healthy — waiting 30s..." repeated → instance launched but never passed the health check
#   "0/0 target(s) healthy"                          → no instance ever registered (launch failure)

# Did the ASG manage to launch an instance?
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name $STACK-dlpod-asg \
  --query "Activities[*].[StartTime,StatusCode,StatusMessage]" --output table --region $REGION

# Was the UserData built correctly? Expect "bootstrap.json N bytes b64, part 1 = M bytes" and part 2
aws logs tail /aws/lambda/$STACK-dlpod-bootstrap-builder --since 1h --region $REGION

# Current target health (only useful if the stack was created with --disable-rollback)
aws elbv2 describe-target-health --target-group-arn $DLPOD_TG \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Reason,TargetHealth.Description]" \
  --output table --region $REGION
```

**Solution:** By default the stack rolls back and deletes the DLPoD instance, so re-create with
`--disable-rollback` to keep it for inspection. Then:

| Finding | Cause | Solution |
|---|---|---|
| Scaling activity `Failed` with `InvalidAMIID.NotFound` / `Unsupported` | The DLPoD AMI is not shared to this account, or was shared for a different region. DLPoD is not a Marketplace product (unlike the AIG AMI) — it is shared privately from the Netskope console | In the Netskope tenant: **Security Cloud Platform → On-Premises Infrastructure → Setup DLP On Demand → AWS → Share Image**, enter this AWS account ID and region, then pass the shared image's ID as `DlpodAmiId` (it appears under **EC2 → AMIs → Private images**). See the [DLP On Demand config guide](https://docs.netskope.com/en/dlpondemandconfig). |
| Scaling activity `Failed` with `InsufficientInstanceCapacity` / `VcpuLimitExceeded` | No capacity or quota for `DlpodInstanceType` (default `c5a.4xlarge`) | Change `DlpodInstanceType` or request a quota increase |
| `<stack>-dlpod-cert-key` fields (`ca_cert_pem`, `leaf_cert_pem`, `leaf_key_pem`) are the literal string `pending` | Cert generator did not overwrite the placeholder secret, so `bootstrap.json` carries `pending` instead of PEM material and `nsbootstrap` cannot bring up TLS on 443 (the builder itself succeeds — all three keys exist) | See [Certificate Issues](#certificate-issues) |
| Target `unhealthy` — `Target.Timeout` | Appliance still booting/bootstrapping; or ALB SG → instance SG blocked | Wait, then re-check; verify `DlpodSecurityGroup` allows 443 from `DlpodAlbSecurityGroup` |
| Target `unhealthy` — `Target.FailedHealthChecks` for a long time | `nsbootstrap` did not complete (bad license key, cannot reach Netskope) | Verify `DlpodLicenseKey`; check the NAT Gateway is `available` and the private route table has `0.0.0.0/0` → NAT; check appliance-side bootstrap status per the Netskope DLP On Demand documentation |
| Gate log shows targets healthy just after the 840 s mark | Slow AMI boot on this instance type | Retry the create; consider a larger `DlpodInstanceType`. The 840 s budget is set by the gate's `TimeoutSeconds` property, bounded by the Lambda's 900 s maximum |

Once the root cause is fixed, delete the failed stack and re-create it. The gate only runs on
`Create` — it never blocks stack updates or later scale-outs.

---

### Issue: DLPoD instance never becomes healthy (after stack creation)

**Cause:** A replacement or scaled-out DLPoD instance (which the readiness gate does not cover)
boots but never passes the ALB health check. Because the ASG uses ELB health checks with a
30-minute grace period, the instance will be terminated and replaced in a loop until the root
cause is fixed.

**Diagnosis:**
```bash
# Instance states and launch/terminate churn
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-dlpod-asg \
  --query "AutoScalingGroups[0].Instances[*].[InstanceId,LifecycleState,HealthStatus,LaunchTemplate.Version]" \
  --output table --region $REGION
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name $STACK-dlpod-asg --max-items 10 \
  --query "Activities[*].[StartTime,StatusCode,Description,Cause]" --output table --region $REGION

# Target health with reason codes
aws elbv2 describe-target-health --target-group-arn $DLPOD_TG \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Reason,TargetHealth.Description]" \
  --output table --region $REGION

# Confirm the launch template UserData is still populated (should be a long base64 string, not empty)
aws ec2 describe-launch-template-versions --launch-template-name $STACK-dlpod-lt --versions '$Latest' \
  --query "LaunchTemplateVersions[0].LaunchTemplateData.UserData" --output text --region $REGION | wc -c
```

**Common causes:**

| Finding | Cause | Solution |
|---|---|---|
| `initial` | ALB just saw the target; health check in progress | Wait — allow up to 10 minutes from launch |
| `unhealthy` - `Target.Timeout` for < 10 min | Appliance still booting and running `nsbootstrap` | Wait; DLPoD needs several minutes before it listens on 443 |
| `unhealthy` - `Target.Timeout` for > 10 min | `DlpodSecurityGroup` no longer allows 443 from `DlpodAlbSecurityGroup`, or `DlpodAlbSecurityGroup` egress was changed | Restore the SG rules from the template |
| `unhealthy` - `Target.FailedHealthChecks` | HTTPS is up but returning 5xx — DLP service not fully initialized or failed to license | Confirm the `DlpodLicenseKey` supplied at stack creation is the correct key for this tenant (a wrong key requires redeploying the stack); check outbound reachability via the NAT Gateway |
| UserData length is `0`/tiny after a stack update | A stack update re-ran `DlpodBootstrapPart1/2` and the builder failed | Check `/aws/lambda/<stack>-dlpod-bootstrap-builder`; fix the secret it could not read and update the stack again |
| Instance terminates every ~30 min with `ELB health check failed` | Any of the above left unresolved | Fix the root cause; the next replacement will succeed |

---

### Issue: DLPoD ALB target healthy but AIG reports DLP errors

**Cause:** The DLPoD appliance is listening and passes the `/` health check, but the AI Gateway's
TLS validation of `https://dlp.aigw.internal` fails or DLP profiles have not been received from
the Netskope management plane.

**Diagnosis:**
```bash
# The CA the AIG trusts (from SSM) must be the one that signed the leaf cert in the DLPoD UserData
aws ssm get-parameter --name /$STACK/dlpod-cert --query Parameter.Value --output text --region $REGION \
  | openssl x509 -noout -subject -issuer -dates
aws secretsmanager get-secret-value --secret-id $STACK-dlpod-cert-key \
  --query SecretString --output text --region $REGION \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['leaf_cert_pem'])" \
  | openssl x509 -noout -subject -issuer -dates
```

The leaf `issuer` must equal the CA `subject` (`CN=dlp.aigw.internal CA`), and neither cert may
be expired (365-day validity from stack creation). If they do not match, see
[AIG cannot verify DLPoD TLS certificate](#issue-aig-cannot-verify-dlpod-tls-certificate). If the
certs are consistent, verify in the Netskope portal that the DLP On Demand appliance shows as
connected and has DLP profiles assigned; consult the Netskope DLP On Demand documentation for
appliance-side checks.

---

## Certificate Issues

### Issue: DLPoD cert missing from SSM

**Symptom:** `/<stack>/dlpod-cert` is absent or still holds the literal value `pending`.

**Cause:** The `DlpodAlbCertificate` custom resource (`CertGeneratorFunction`, Lambda
`<stack>-certgen`) failed during stack creation. The template creates `/<stack>/dlpod-cert` with
the placeholder value `pending` and `<stack>-dlpod-cert-key` with `pending` fields; the Lambda
overwrites both. If it fails, the stack normally rolls back because the DLPoD listener,
`DlpodBootstrapPart1/2`, and everything downstream depend on it.

**Diagnosis:**
```bash
# Check CloudFormation events for the cert generator resource
aws cloudformation describe-stack-events --stack-name $STACK --region $REGION \
  --query "StackEvents[?LogicalResourceId=='DlpodAlbCertificate'].[Timestamp,ResourceStatus,ResourceStatusReason]" \
  --output table

# Check cert generator Lambda logs — success is "Imported leaf cert ... wrote CA cert to SSM ..."
# followed by "Wrote CA+leaf cert+key to ..."
aws logs tail /aws/lambda/$STACK-certgen --since 60m --region $REGION

# Confirm the placeholder was overwritten
aws ssm get-parameter --name /$STACK/dlpod-cert --query Parameter.Value --output text --region $REGION | head -1
#   expected: -----BEGIN CERTIFICATE-----   (not "pending")
```

**Common causes:**

| Log pattern | Cause | Solution |
|---|---|---|
| `CalledProcessError` from an `openssl` step | openssl invocation failed in the Lambda runtime | Inspect the full traceback; the template relies on the `openssl` binary present in the `python3.12` runtime |
| `AccessDenied` on `acm:ImportCertificate` | Cert generator Lambda role missing ACM permissions | Check `<stack>-certgen-role` policy (`ImportCert` statement) |
| `AccessDenied` on `ssm:PutParameter` | Cert generator Lambda role missing SSM permissions | Check `<stack>-certgen-role` policy (`WriteCertParam` statement, scoped to `/<stack>/*`) |
| `AccessDenied` on `secretsmanager:PutSecretValue` | Role missing access to `<stack>-dlpod-cert-key` | Check `<stack>-certgen-role` policy (`WriteCertKeySecret` statement) |
| `ParameterAlreadyExists` on `DlpodCertParameter` | Leftover parameter from a previous stack with the same name | Delete `/<stack>/dlpod-cert` manually, then re-create the stack |

**Impact:** If the CA PEM in SSM is still `pending`, every AIG activation Lambda run writes
`"certificate": "pending"` into the bootstrap secret, and AIG instances cannot validate the DLPoD
endpoint at enrollment. If `<stack>-dlpod-cert-key` is still `pending`, the DLPoD `bootstrap.json`
contains invalid cert material and DLPoD never becomes healthy. Fix the cert generator issue and
re-create the stack (or update it in a way that re-runs `DlpodAlbCertificate`).

---

### Issue: AIG cannot verify DLPoD TLS certificate

**Cause:** The CA cert PEM the AIG received in its bootstrap secret does not match the CA that
signed the leaf cert the DLPoD ALB presents. The activation Lambda reads `/<stack>/dlpod-cert`
from SSM at each AIG instance launch and writes the `dlp` block into `<stack>-aig-bootstrap` then;
the DLPoD ALB listener uses the ACM cert imported by `DlpodAlbCertificate`. Both originate from the
same `CertGeneratorFunction` run, so a mismatch means one side is stale: the SSM parameter was
edited, the ACM cert was replaced, or the AIG instance enrolled before a stack update regenerated
the hierarchy.

**Diagnosis:**
```bash
# Check the dlp block in the bootstrap secret (host and first line of the cert)
aws secretsmanager get-secret-value \
  --secret-id $STACK-aig-bootstrap \
  --query SecretString --output text --region $REGION | \
  python3 -c "import json,sys; d=json.load(sys.stdin).get('dlp',{}); print(d.get('host')); print(d.get('certificate','MISSING')[:40])"

# Compare the CA in SSM with the CA in the bootstrap secret
aws ssm get-parameter --name /$STACK/dlpod-cert --query Parameter.Value --output text --region $REGION \
  | openssl x509 -noout -fingerprint -sha256
aws secretsmanager get-secret-value --secret-id $STACK-aig-bootstrap --query SecretString --output text --region $REGION \
  | python3 -c "import json,sys; print(json.load(sys.stdin)['dlp']['certificate'])" \
  | openssl x509 -noout -fingerprint -sha256

# Confirm the cert the DLPoD ALB listener serves chains to that CA
ALB_CERT=$(aws elbv2 describe-listeners \
  --load-balancer-arn $(aws elbv2 describe-load-balancers --names $STACK-dlpod-alb --query "LoadBalancers[0].LoadBalancerArn" --output text --region $REGION) \
  --query "Listeners[0].Certificates[0].CertificateArn" --output text --region $REGION)
aws acm get-certificate --certificate-arn $ALB_CERT --query Certificate --output text --region $REGION \
  | openssl x509 -noout -subject -issuer -dates
```

The listener cert's `issuer` must be `CN=dlp.aigw.internal CA` and the two SHA-256 fingerprints
must match. Expected host in the bootstrap secret is `https://dlp.aigw.internal`.

**Solution:** The bootstrap secret is rewritten at every AIG launch, so terminate the affected AIG
instance (or start an instance refresh on `<stack>-aig-asg`) and let the ASG replace it — the new
instance's activation run picks up the current SSM value. If SSM and ACM themselves disagree,
re-run the cert generator via a stack update that touches `DlpodAlbCertificate`, then refresh both
the DLPoD and AIG ASGs so DLPoD serves the new leaf and AIG trusts the new CA. Certificates are
valid for 365 days from generation; an expired CA produces the same symptom and requires the same
regenerate-and-refresh procedure.

---

## AI Guardrails Issues

Applies only when the stack was created with `GuardrailsImageS3Bucket` set.

**How Guardrails comes up:** the Guardrails ASG has no lifecycle hook and uses `HealthCheckType: EC2`
(no grace period), so an instance is never replaced for failing the ALB health check. The launch
template UserData does all the work at first boot: it runs `nvidia-smi`, mounts the local NVMe
instance store at `/opt/dlami/nvme` (g4dn/g5 only), installs Docker and the NVIDIA Container
Toolkit if missing, moves the Docker data-root onto the NVMe, downloads `aisecurity-llm.tgz` from
S3 to the NVMe, `docker load`s it, starts the `guardrails` container, and polls
`http://localhost:8080/ping` up to 90 times at 10 s intervals (15 minutes). There are no
CloudWatch logs for this tier; the record is `/var/log/user-data.log` on the instance.

The stack-creation gate `GuardrailsReadinessGate` is an `AWS::CloudFormation::WaitCondition`
(`Timeout: 3600`, `Count: 1`) that the first Guardrails instance satisfies with a cfn-signal-style
`curl -X PUT` to the `GuardrailsWaitHandle` URL once its local `/ping` returns 200. It does not use
the DLPoD readiness Lambda, does not poll the ALB target group, and writes nothing to
`/aws/lambda/<stack>-dlpod-readiness`.

### Issue: Stack fails at `GuardrailsReadinessGate` — container not healthy

**Symptom:** the `GuardrailsReadinessGate` stack event is `CREATE_FAILED` with one of:

| Event reason | Meaning |
|---|---|
| `WaitCondition received failed message: 'FAILURE' for uniqueId: <hostname>` with reason `Container not healthy after 15 min` | UserData ran to the end but `/ping` never returned 200 within 90 × 10 s |
| `WaitCondition received failed message: 'FAILURE' ...` with reason `UserData script failed` | A command in UserData exited non-zero (`bash -e` + `trap ... ERR`) — `nvidia-smi`, `mkfs`, `apt-get`, `aws s3 cp`, `docker load`, or `docker run` |
| `Failed to receive 1 resource signal(s) within the specified duration` | No signal at all in 60 minutes — the instance never launched, never reached UserData, or could not reach the S3 pre-signed handle URL (no NAT route) |

**Cause:** in order of likelihood: the S3 download or `docker load` failed (wrong bucket or key,
bucket in another region, `GuardrailsRole` cannot read the object, no NAT route); the instance
fell back to the gp3 root volume instead of the NVMe instance store and the download plus
`docker load` plus model load exceeded 15 minutes; `nvidia-smi` failed because `GuardrailsAmiId` is
not a Deep Learning Base GPU AMI; or the ASG could not launch the instance (see
[`VcpuLimitExceeded`](#issue-guardrails-asg-launch-fails-with-vcpulimitexceeded)).

**Diagnosis:** by default the stack rolls back and terminates the instance, so re-create with
`--disable-rollback` to keep it for inspection.

```bash
# Exact failure reason as CloudFormation recorded it
aws cloudformation describe-stack-events --stack-name $STACK --region $REGION \
  --query "StackEvents[?LogicalResourceId=='GuardrailsReadinessGate'].[Timestamp,ResourceStatus,ResourceStatusReason]" \
  --output table

# Did an instance launch at all?
aws autoscaling describe-scaling-activities --auto-scaling-group-name $STACK-guardrails-asg \
  --query "Activities[*].[StartTime,StatusCode,StatusMessage]" --output table --region $REGION
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names $STACK-guardrails-asg \
  --query "AutoScalingGroups[0].Instances[*].InstanceId" --output text --region $REGION

# Open a shell on it (GuardrailsRole includes AmazonSSMManagedInstanceCore)
aws ssm start-session --target <instance-id> --region $REGION
```

On the instance:

| Command | What to look for |
|---|---|
| `sudo cat /var/log/user-data.log` | Full UserData trace: `nvidia-smi` output, NVMe mount, `aws s3 cp`, `docker load`, `docker run`, the `/ping` poll, and the `signal_cfn` call |
| `nvidia-smi` | Driver present and GPU visible; failure means the AMI is not a GPU/Deep Learning image |
| `lsblk -dpno NAME,MODEL` | A device whose model is `Amazon EC2 NVMe Instance Storage` must be present (g4dn/g5) |
| `mountpoint /opt/dlami/nvme` | `is a mountpoint` — the tarball and Docker data-root are on the instance store |
| `sudo docker info \| grep 'Docker Root Dir'` | Expect `/opt/dlami/nvme/docker`. If it is under `/tmp`, the NVMe path was not used and everything ran on the gp3 root — much slower boot |
| `sudo docker ps -a` | Is `guardrails` `Up` or `Exited`? |
| `sudo docker logs guardrails` | Model load progress, CUDA errors, port binding errors |
| `curl -s http://localhost:8080/ping` | `Healthy` once the container is ready |

**Solution:**

| Finding | Solution |
|---|---|
| `aws s3 cp` fails (`NoSuchKey`, `AccessDenied`, `301`) | Fix `GuardrailsImageS3Bucket` / `GuardrailsImageS3Key`; the bucket must be readable by `<stack>-guardrails-role`. A bucket in a different region than the stack also slows the multi-GB download — keep it in the same region |
| `Docker Root Dir` under `/tmp`, or no `NVMe Instance Storage` device | The instance type has no local instance store. Keep `GuardrailsInstanceType` at one of the allowed g4dn/g5 types; do not widen `AllowedValues` to EBS-only types |
| Container `Up` and `/ping` returns `Healthy`, but the signal arrived after 15 minutes | Slow path: NVMe not used, cross-region bucket, or small instance. Fix the above; if still marginal, move to `g5.xlarge` |
| `nvidia-smi` fails | Use an AWS Deep Learning Base GPU AMI (Ubuntu 22.04) for the region as `GuardrailsAmiId` |
| No instance launched | See [`VcpuLimitExceeded`](#issue-guardrails-asg-launch-fails-with-vcpulimitexceeded) |

Once fixed, delete the failed stack and re-create it. The WaitCondition only gates the initial
create; later scale-outs and replacements are not gated and are visible only through the
Guardrails target group health.

---

### Issue: Create fails immediately with `GuardrailsAmiId is required when GuardrailsImageS3Bucket is set`

**Cause:** the template `Rules` assertion `GuardrailsAmiRequired` rejected the parameter set before
any resource was created: `GuardrailsImageS3Bucket` is non-empty but `GuardrailsAmiId` is empty.

**Diagnosis:** the `create-stack` call itself returns the assertion text; no stack is created.

**Solution:** pass a Deep Learning Base GPU AMI ID for the region as `GuardrailsAmiId`
(see [docs/DEPLOYMENT.md](DEPLOYMENT.md), AI Guardrails prerequisites), or leave
`GuardrailsImageS3Bucket` empty to skip the tier.

---

### Issue: Guardrails ASG launch fails with `VcpuLimitExceeded`

**Cause:** no quota for "Running On-Demand G and VT instances" in the region; `g4dn.xlarge` needs 4 vCPU.

**Diagnosis:**
```bash
aws autoscaling describe-scaling-activities --auto-scaling-group-name $STACK-guardrails-asg \
  --query "Activities[?StatusCode=='Failed'].[StartTime,StatusMessage]" --output table --region $REGION
aws service-quotas get-service-quota --service-code ec2 --quota-code L-DB2E81BA \
  --query "Quota.Value" --output text --region $REGION     # Running On-Demand G and VT instances
```

**Solution:** request at least 4 vCPU (more for `g4dn.2xlarge`/`g5.2xlarge` or a higher
`GuardrailsDesiredCapacity`) in Service Quotas → Amazon EC2, then delete and re-create the stack.

---

### Issue: AIG enrolled but Guardrails shows unlinked / inspection not applied

**Cause:** AIG validates `ai_guardrails.host` at enrollment; if it could not reach
`http://guardrails.aigw.internal:8080/invocations` at that moment the appliance enrolls in an
"unlinked" state and does not retry. After a successful create this happens when Guardrails
targets later became unhealthy, or when an AIG instance enrolled while all Guardrails targets were down.

**Diagnosis:**
```bash
# Does the bootstrap secret carry the guardrails block?
aws secretsmanager get-secret-value --secret-id $STACK-aig-bootstrap \
  --query SecretString --output text --region $REGION | \
  python3 -c "import json,sys; print(json.load(sys.stdin).get('ai_guardrails','MISSING'))"

# Are Guardrails targets healthy?
GR_TG=$(aws elbv2 describe-target-groups --query "TargetGroups[?contains(TargetGroupName,'$STACK-guardrails')].TargetGroupArn" \
  --output text --region $REGION)
aws elbv2 describe-target-health --target-group-arn $GR_TG \
  --query "TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Reason]" --output table --region $REGION

# From an AIG instance (aws ssm start-session --target <aig-instance-id>):
curl -s http://guardrails.aigw.internal:8080/ping      # expect: Healthy
```

**Solution:** if `ai_guardrails` is `MISSING`, the stack was created without Guardrails (or the
Activation Lambda has not run since) — see the Activation Lambda logs. If targets are unhealthy,
fix the Guardrails instance first (previous issue; because the ASG uses EC2 health checks it will not
self-replace — terminate the instance manually to get a fresh one). Then terminate the affected AIG
instance so the ASG replaces it and it re-enrolls with the Guardrails host reachable.

---

## Stack Issues

### Issue: Stack stuck at `CREATE_IN_PROGRESS` for more than 30 minutes

**Cause:** A custom resource or Lambda is not responding to CloudFormation.

**Diagnosis:**
```bash
# Find the stuck resource
aws cloudformation describe-stack-events --stack-name $STACK --region $REGION \
  --query "StackEvents[?ResourceStatus=='CREATE_IN_PROGRESS'].[LogicalResourceId,ResourceType,Timestamp]" \
  --output table

# If the stuck resource is a custom resource, check the matching Lambda's logs
aws logs tail /aws/lambda/$STACK-certgen --since 60m --region $REGION                  # DlpodAlbCertificate / AigAlbCertificate
aws logs tail /aws/lambda/$STACK-dlpod-bootstrap-builder --since 60m --region $REGION  # DlpodBootstrapPart1 / Part2
aws logs tail /aws/lambda/$STACK-dlpod-readiness --since 60m --region $REGION          # DlpodReadinessGate only
```

**Common causes:**

| Stuck resource | Cause | Solution |
|---|---|---|
| `DlpodAlbCertificate` or `AigAlbCertificate` | Cert generator Lambda failed and did not send a response to CloudFormation | CloudFormation waits up to 1 hour for a custom resource response; check `<stack>-certgen` logs; the stack rolls back after timeout |
| `DlpodBootstrapPart1` / `DlpodBootstrapPart2` | Bootstrap builder Lambda failed without responding (rare — it catches exceptions and reports `FAILED`) | Check `<stack>-dlpod-bootstrap-builder` logs; verify the Lambda was invoked at all |
| `DlpodReadinessGate` (up to 840 s) | Normal — the readiness Lambda is polling DLPoD target health | Watch `<stack>-dlpod-readiness` logs for `N/M target(s) healthy`; if it fails see [DLP On Demand Issues](#dlp-on-demand-issues) |
| `GuardrailsReadinessGate` (up to 60 min) | Normal for the first ~15 minutes — the `WaitCondition` is waiting for the first Guardrails instance's UserData to signal. It has no Lambda and no CloudWatch logs | `aws ssm start-session` to the Guardrails instance and read `/var/log/user-data.log`; see [AI Guardrails Issues](#ai-guardrails-issues). Beyond 15 minutes with no `CREATE_FAILED` the instance never reached the signal step |
| `GatewayAutoScalingGroup` | ASG waiting for the AIG launch lifecycle hook to complete | Check AIG activation Lambda logs |
| `DlpodAutoScalingGroup` | Instance launch failing (capacity, AMI) | `aws autoscaling describe-scaling-activities --auto-scaling-group-name $STACK-dlpod-asg`; there is no lifecycle hook on this ASG |

**Prevention — dry-run before creating:** a `CREATE` change set validates the template, parameters,
`Rules` assertions, and the caller's IAM permissions without creating resources. It leaves an empty
`REVIEW_IN_PROGRESS` stack shell under the scratch name, so delete that afterwards.

```bash
aws cloudformation create-change-set --stack-name $STACK-dry-run --change-set-name dry-run \
  --change-set-type CREATE \
  --template-url https://<bucket>.s3.$REGION.amazonaws.com/templates/gateway-combined.yaml \
  --parameters ParameterKey=NetskopeTenantUrl,ParameterValue=https://<tenant>.goskope.com \
    ParameterKey=NetskopeApiToken,ParameterValue=$NETSKOPE_API_TOKEN \
    ParameterKey=DlpodLicenseKey,ParameterValue=$DLPOD_LICENSE_KEY \
  --capabilities CAPABILITY_NAMED_IAM --region $REGION
aws cloudformation describe-change-set --stack-name $STACK-dry-run --change-set-name dry-run \
  --query "[Status,StatusReason]" --output text --region $REGION      # CREATE_COMPLETE = template and parameters accepted
aws cloudformation delete-stack --stack-name $STACK-dry-run --region $REGION
```

---

### Issue: Stack stuck in `ROLLBACK_FAILED` or `DELETE_FAILED`

**Cause:** a resource could not be deleted during rollback or deletion. Typical blockers in this
template: an AIG instance held in `Terminating:Wait` (next issue), an ENI still attached to the
NAT Gateway or an ALB that has not finished deleting, or a resource that was modified outside
CloudFormation.

**Diagnosis:**
```bash
aws cloudformation describe-stack-events --stack-name $STACK --region $REGION \
  --query "StackEvents[?ResourceStatus=='DELETE_FAILED'].[LogicalResourceId,ResourceStatusReason]" --output table
```

**Solution:** fix or manually remove the blocking resource, then retry the delete. If it still
fails, force-delete the stack (available in AWS CLI v2 since 2024; resources that still cannot be
deleted are left behind and must be cleaned up by hand — check `/<stack>/dlpod-cert`, ACM
certificates, and the private hosted zone afterwards).

```bash
aws cloudformation delete-stack --stack-name $STACK --region $REGION
# If that also fails:
aws cloudformation delete-stack --stack-name $STACK --deletion-mode FORCE_DELETE_STACK --region $REGION
```

---

### Issue: Stack deletion hangs

**Cause:** An AIG instance is held in `Terminating:Wait` by the AIG termination lifecycle hook
(`<stack>-aig-terminate-hook`, 120 s heartbeat, `DefaultResult: CONTINUE`). Only the AIG ASG has
lifecycle hooks — DLPoD and Guardrails instances terminate immediately.

**Diagnosis:**
```bash
# Check for AIG instances still in Terminating:Wait
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names $STACK-aig-asg \
  --query "AutoScalingGroups[0].Instances[?LifecycleState=='Terminating:Wait'].[InstanceId,LifecycleState]" \
  --output table --region $REGION
```

**Solution:** The hook defaults to `CONTINUE` after 2 minutes even if the activation Lambda fails,
so this state is transient. If an instance stays there longer (for example because the SNS
subscription was deleted before the ASG), force-complete the lifecycle action:

```bash
INSTANCE_ID=<stuck-instance-id>

aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name $STACK-aig-terminate-hook \
  --auto-scaling-group-name $STACK-aig-asg \
  --lifecycle-action-result CONTINUE \
  --instance-id $INSTANCE_ID \
  --region $REGION
```

Deleting the stack also deletes `<stack>-dlpod-cert-key`, `/<stack>/dlpod-cert`, and the ACM
certificate imported by the cert generator. The `/aig/<stack>/<instance-id>` parameters are not
stack resources — the activation Lambda deletes each one during the termination hook. If that
Lambda failed (or the SNS subscription was already gone), the parameter lingers and the appliance
may remain registered in the Netskope tenant; delete the parameter with `aws ssm delete-parameter`
and remove the appliance in the portal.

Stack deletion typically completes 8–15 minutes after the `delete-stack` command, dominated by
NAT Gateway and VPC deletion.

---

## Log Patterns Reference

### AIG Activation Lambda (`/aws/lambda/<stack>-aig-activation`)

The Lambda logs with plain `print`, so lines have no `[INFO]` prefix — just the Lambda
`START` / `END` / `REPORT` records and the messages below.

**Successful launch:**
```
Registered appliance <id> for i-abc123 (dlp=yes, guardrails=no), completing CONTINUE
```
`guardrails=yes` appears when the stack was created with `GuardrailsImageS3Bucket`.

**Successful termination:**
```
Deregistered appliance <id> for i-abc123
```

**Failure patterns** (a Python traceback, followed by `CONTINUE`/`ABANDON` via the lifecycle API):
```
urllib.error.HTTPError: HTTP Error 401: Unauthorized        # bad or expired NetskopeApiToken
urllib.error.HTTPError: HTTP Error 403: Forbidden           # token lacks AIG Administrator role
urllib.error.URLError: <urlopen error [Errno -2] ...>       # tenant URL unresolvable / wrong
TimeoutError / socket.timeout                               # Netskope API unreachable (30 s limit)
botocore.exceptions.ClientError: ... ParameterNotFound ...  # /<stack>/dlpod-cert missing
KeyError: 'id'                                              # unexpected API response body
```
On launch a traceback means the instance was ABANDONED; on termination the traceback is logged
and the hook still completes with `CONTINUE`.

---

### Cert Generator Lambda (`/aws/lambda/<stack>-certgen`)

Runs only when `DlpodAlbCertificate` (and `AigAlbCertificate`, if `AcmCertificateArn` is empty)
is created, updated, or deleted.

**Successful create:**
```
[INFO] Imported leaf cert arn:aws:acm:...:certificate/... to ACM, wrote CA cert to SSM /<stack>/dlpod-cert
[INFO] Wrote CA+leaf cert+key to arn:aws:secretsmanager:...:secret:<stack>-dlpod-cert-key-...
```
The second line is absent for the AIG ALB cert (no key secret is passed for it).

**Failure pattern:**
```
[ERROR] CertGeneratorFunction failed
Traceback (most recent call last): ...
```
followed by a `FAILED` response to CloudFormation and a stack rollback.

**On delete:** `Deleted cert arn:aws:acm:...` or `Could not delete cert ...` (a warning — the
stack delete still succeeds).

---

### DLPoD Bootstrap Builder Lambda (`/aws/lambda/<stack>-dlpod-bootstrap-builder`)

Runs twice at stack creation (once per `DlpodBootstrapPart1` / `Part2`) and again on any update
that changes their properties.

**Successful create:**
```
[INFO] bootstrap.json 5xxx bytes b64, part 1 = 2xxx bytes
[INFO] bootstrap.json 5xxx bytes b64, part 2 = 2xxx bytes
```
The two part sizes should sum to the total. Exact byte counts vary with the certificate material.

**Failure pattern:**
```
[ERROR] DlpodBootstrapBuilderFunction failed
botocore.exceptions.ClientError: ... AccessDeniedException ...   # cannot read a secret
```

A cert-key secret that is still the placeholder does **not** fail the builder: all three keys
(`ca_cert_pem`, `leaf_cert_pem`, `leaf_key_pem`) exist with the value `pending`, so the builder
logs a normal success with a small byte count (a few hundred bytes instead of several thousand)
and DLPoD boots with unusable TLS material. In practice this cannot occur on create because
`DlpodBootstrapPart1/2` `DependsOn: DlpodAlbCertificate`; a suspiciously small
`bootstrap.json N bytes` after a stack update points at the cert generator.

---

### Readiness Gate Lambda (`/aws/lambda/<stack>-dlpod-readiness`)

Used only by `DlpodReadinessGate`. `GuardrailsReadinessGate` is a CloudFormation `WaitCondition`
signalled from the Guardrails instance's UserData and produces no CloudWatch logs — its record is
`/var/log/user-data.log` on the instance and the `GuardrailsReadinessGate` stack event.

**Successful gate:**
```
[INFO] RequestType: Create
[INFO] 0/1 target(s) healthy — waiting 30s...
[INFO] 0/1 target(s) healthy — waiting 30s...
...
[INFO] All 1 target(s) healthy
```

**Failed gate** (after 840 s of polling):
```
[INFO] 0/1 target(s) healthy — waiting 30s...      # repeated ~28 times
```
followed by the CloudFormation reason `Targets in <stack>-dlpod-tg did not become healthy within
840s`. `0/0 target(s)` throughout means no instance ever registered with the target group
— check the ASG scaling activities for launch failures.

**Update / delete:**
```
[INFO] RequestType: Update      (or Delete)
```
and an immediate `SUCCESS` — the gate never polls outside of `Create`.

---

### DLPoD Bootstrap Timeline (per instance)

There are no per-instance logs for DLPoD in CloudWatch. Use the DLPoD ALB target health as the
progress indicator:

| Phase | Observable state | Typical duration |
|---|---|---|
| Instance launch | ASG instance `Pending` → `InService`; target `initial` | <1 minute |
| First boot, `nsbootstrap.service` applies `bootstrap.json` | Target `unhealthy` — `Target.Timeout` (nothing listening on 443) | 3–8 minutes |
| DLP service starts, licenses, connects to Netskope | Target `unhealthy` — `Target.FailedHealthChecks`, then `healthy` after 2 consecutive passes at 30 s | 1–3 minutes |
| **Total to healthy** | | **5–10 minutes** |
| ASG replacement threshold | ELB health check grace period | 30 minutes |
| Stack-creation gate | `DlpodReadinessGate` fails the stack if not healthy | 840 s (14 minutes) |

For appliance-side bootstrap status, consult the Netskope DLP On Demand documentation.

---

### Guardrails Boot Timeline (per instance, `/var/log/user-data.log`)

No CloudWatch logs. Read `/var/log/user-data.log` via `aws ssm start-session`; the phases appear in this order:

| Phase | Log evidence | Typical duration (NVMe path) |
|---|---|---|
| Driver check | `nvidia-smi` table | seconds |
| NVMe mount | `mountpoint` / `mkfs.ext4` / `mount ... /opt/dlami/nvme` — absent if the DLAMI pre-mounted it; `NVME=/tmp` means fallback to the gp3 root | seconds |
| Docker + NVIDIA toolkit install (if the AMI lacks them) | `apt-get install ... docker-ce` / `nvidia-container-toolkit`, `systemctl restart docker` | 1–3 minutes |
| Image download and load | `aws s3 cp s3://.../aisecurity-llm.tgz`, `docker load`, `Using image: <repo:tag>` | 2–6 minutes (longer for a cross-region bucket or `/tmp` fallback) |
| Container start and model load | `docker run -d --name guardrails ...`, then repeated `curl` attempts | 2–5 minutes |
| Signal | `Guardrails container healthy on port 8080` then the `signal_cfn SUCCESS "Container healthy"` PUT | — |
| Failure | `ERROR: Guardrails not healthy after 15 minutes` + `signal_cfn FAILURE "Container not healthy after 15 min"`, or `UserData script failed` from the `ERR` trap | at 15 minutes / on first failed command |
| Stack-creation gate | `GuardrailsReadinessGate` WaitCondition times out with `Failed to receive 1 resource signal(s) within the specified duration` if no signal arrives | 60 minutes |
