# Security Operations & Recovery — Lab Evidence

**Lab:** Triage prepared security findings and inspect CloudTrail evidence from resource changes.
**Account/region:** AWS account (ID redacted), `us-east-1`
**Date:** 2026-09-04


## Training resource change + CloudTrail evidence

**Action taken:** Added tag `lab=secops-triage` to S3 bucket `secops-lab-bucket` via the console.

| Field | Value |
|---|---|
| Event name | `TagResource` |
| Event source | `s3.amazonaws.com` |
| Event time (UTC) | `2026-09-04T07:40:59Z` |
| Principal type | IAM User |
| Principal | `arn:aws:iam::<ACCOUNT_ID>:user/<redacted-username>` |
| MFA authenticated | `false` |
| Region | `us-east-1` |
| Resource | `arn:aws:s3:::secops-lab-bucket` (`AWS::S3::Bucket`) |
| Tag applied | Key = `lab`, Value = `secops-triage` |
| HTTP status | `204` (success) |
| Read-only event | `false` (management/write event) |
| Event ID | `28518615-d481-4ac9-98ad-38270a416c20` |


## Access Analyzer finding

**Source:** Live analyzer (account-level, default zone of trust)

| Field | Value |
|---|---|
| Finding ID | `aab1425a-6470-4282-b8dd-9b28ce69dd50` |
| Resource | `arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsDeployRole` |
| Resource type | IAM Role |
| External principal | Federated (OIDC) — `arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com` |
| Action | `sts:AssumeRoleWithWebIdentity` |
| Access level | Write |
| Condition | None applied |
| Resource control policy (RCP) restriction | Not applicable |
| Status | Active |


---

## WAF sample rule scenario

No WAF WebACL was deployed live (WAF has no free tier — it bills per Web ACL per hour from creation). The scenarios below are representative sample requests against a hypothetical WebACL protecting a login/API endpoint, used to practice the allow/block/count decision.

| # | Request scenario | Decision | Reason |
|---|---|---|---|
| 1 | Query string contains `' OR '1'='1` against a search parameter | **Block** | Matches classic SQL injection signature (AWS Managed Rule `SQLiRuleSet`); high-confidence malicious pattern, no legitimate use case |
| 2 | Single GET from a brand-new IP, normal headers, no payload anomalies, first time seen | **Count** | No malicious signal yet; count to build a baseline/rate profile before deciding to block, avoids false-positive blocking of a new legitimate user |
| 3 | 800 requests/5 min from one IP to `/login`, no successful auths | **Block** (rate-based rule) | Pattern matches credential-stuffing/brute-force; volume and failure rate justify blocking regardless of payload content |
| 4 | Request from a known corporate NAT IP (on allow-list) that trips a generic rate-based rule during a load test | **Allow** (scope-down / allow-list exception) | Source is trusted and behavior is explained (internal load test); blocking would be a false positive — better to add an IP-set exception than block |
| 5 | `User-Agent` header missing entirely, request otherwise well-formed, hitting a public read-only GET endpoint | **Count** | Anomalous but not inherently malicious (some legitimate clients/scripts omit UA); count first to see if it correlates with other bad signals before blocking |
| 6 | Request path contains `../../etc/passwd` | **Block** | Matches path traversal / LFI signature (AWS Managed Rule for common exploits); no legitimate request should ever contain this pattern |

---

## Backup sample plan

No live AWS Backup plan was created. The following is a representative sample plan reviewed for the exercise.

| Field | Value |
|---|---|
| Plan name | `daily-s3-backup-plan` |
| Schedule | Daily at `05:00 UTC` (cron: `cron(0 5 * * ? *)`) |
| Retention period | 35 days |
| Lifecycle | No cold-storage transition (retention too short to benefit) |
| Protected resource type | S3 bucket (`secops-lab-bucket`) |
| Backup vault | `Default` |
| Backup rule copy/cross-region | None (single-region for this lab) |



---

## Finding triage worksheet (GuardDuty / Security Hub / Inspector — sample findings)

No live GuardDuty/Security Hub/Inspector trial findings were used; the worksheet below is completed against representative sample findings for each service.

### Worksheet 1 — GuardDuty
| Field | Value |
|---|---|
| Finding source | GuardDuty |
| Finding title/type | `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration.OutsideAWS` |
| Severity | High |
| Affected resource | IAM user (temporary EC2 instance credentials) |
| Summary | Credentials belonging to an EC2 instance role were used from an IP address outside AWS, suggesting the credentials were exfiltrated from the instance and used externally. |
| True/false positive | True positive (pending confirmation) |
| Triage decision | Escalate |
| Remediation/next action | Revoke/rotate the instance role's temporary credentials, isolate the instance, review CloudTrail for actions taken with the credentials, check instance for compromise (e.g. exposed metadata service, SSRF vector) |
| Priority/SLA | Critical — 1 hour |

### Worksheet 2 — Security Hub
| Field | Value |
|---|---|
| Finding source | Security Hub (CIS AWS Foundations control) |
| Finding title/type | `1.14 Ensure hardware MFA is enabled for the root account` (fail) |
| Severity | Medium |
| Affected resource | Root account |
| Summary | Root account does not have hardware (or any) MFA enabled, leaving it protected only by password. |
| True/false positive | True positive |
| Triage decision | Remediate now |
| Remediation/next action | Enable MFA (virtual or hardware) on the root account immediately; confirm root access keys don't exist |
| Priority/SLA | High — same business day |

### Worksheet 3 — Inspector
| Field | Value |
|---|---|
| Finding source | Inspector |
| Finding title/type | `CVE-2024-XXXXX` in `openssl` package on EC2 instance |
| Severity | High (CVSS 8.1) |
| Affected resource | EC2 instance running outdated `openssl` |
| Summary | Inspector detected a vulnerable OpenSSL version with a known remote exploit affecting TLS handling, on an internet-facing instance. |
| True/false positive | True positive |
| Triage decision | Investigate → Remediate |
| Remediation/next action | Patch `openssl` via package manager, redeploy/restart affected service, confirm exposure window via CloudTrail/VPC Flow Logs, re-scan to confirm resolution |
| Priority/SLA | High — 24–48 hours (internet-facing increases urgency) |

---

## Live vs. sample statement

> In this exercise, **CloudTrail**, **S3**, and **IAM Access Analyzer** were exercised live against my own AWS account, generating real evidence at no cost, within Free Tier limits. **GuardDuty, Security Hub, and Inspector** findings were triaged using representative sample data rather than live trial findings, to avoid any risk of trial-expiry billing. **WAF** and **AWS Backup** were reviewed using sample scenarios only and no live resources were created for either — WAF has no free tier at all (billed per Web ACL-hour from creation), and a live Backup job was judged unnecessary to demonstrate the required understanding. Total cost incurred: $0.00.

---
