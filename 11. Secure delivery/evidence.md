# Secure Delivery — GitHub Actions OIDC & Production Change Control — Evidence

**Purpose:** Stand up real OIDC trust between GitHub Actions and AWS, run the workflow, and produce a deployment evidence package for a TypeScript CDK change.
**Repo:** `Kad-19/aws_secure_deilvery_lab` (public)
**Region:** `us-east-1`

---

## 1. OIDC identity provider

| Field | Value |
|---|---|
| Provider ARN | `arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com` |
| Provider status | Reused existing provider already present in the account |
| Audience (`ClientIDList`) | `sts.amazonaws.com` |

---

## 2. IAM role

| Field | Value |
|---|---|
| Role name | `training-cdk-diff-role` |
| Role ARN | `arn:aws:iam::<ACCOUNT_ID>:role/training-cdk-diff-role` |
| Permissions | Inline policy `cdk-diff-readonly` — read-only (`cloudformation:DescribeStacks`, `GetTemplate`, `DescribeStackEvents`, `ListStackResources`, `sts:GetCallerIdentity`) |

**Final trust policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:Kad-19@*/aws_secure_deilvery_lab@*:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

---

## 3. Workflow

| Field | Value |
|---|---|
| Workflow file | `.github/workflows/training-cdk-diff.yml` |
| Trigger | `workflow_dispatch` (manual only) |
| Permissions block | `id-token: write`, `contents: read` |
| Auth method | OIDC via `aws-actions/configure-aws-credentials@v4`, `role-to-assume: ${{ secrets.AWS_ROLE_ARN }}` |
| Role session tagging | Disabled (`role-skip-session-tagging: true`) |
| Successful run URL | https://github.com/Kad-19/aws_secure_deilvery_lab/actions/runs/34121240754 |

---

## 4. Trust policy debugging — root cause and fix

The first several runs failed at the credentials step with:
```
Error: Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

**Investigation ruled out, in order:**
- Wrong branch (confirmed run executed on `main`).
- OIDC provider missing/misconfigured (confirmed provider exists with correct `ClientIDList`).
- Role ARN mismatch between the `AWS_ROLE_ARN` secret and the real role (confirmed identical via length/prefix/suffix check).
- Permissions boundary on the role (none present).
- Role session tagging: `aws-actions/configure-aws-credentials@v4` requests `sts:TagSession` by default, which the trust policy didn't allow. Disabled via `role-skip-session-tagging: true` — this changed the failure mode but did not resolve it, confirming it was a secondary issue, not the root cause.

**Root cause — confirmed by decoding the actual OIDC token issued for the run:**
```json
{
  "aud": "sts.amazonaws.com",
  "repository": "Kad-19/aws_secure_deilvery_lab",
  "repository_owner": "Kad-19",
  "ref": "refs/heads/main",
  "sub": "repo:Kad-19@<owner_id>/aws_secure_deilvery_lab@<repo_id>:ref:refs/heads/main"
}
```

GitHub now embeds the immutable numeric owner ID and repository ID directly into the `sub` claim (`owner@ownerId/repo@repoId`), as a hardening measure against repo-rename or ownership-transfer hijacking of trust relationships. The original trust policy condition (`repo:Kad-19/aws_secure_deilvery_lab:ref:refs/heads/main`) did not include these `@id` segments and therefore never matched the real token — producing a generic `AssumeRoleWithWebIdentity` denial with no further detail.

**Fix:** updated the `sub` condition to wildcard the numeric ID segments:
```
repo:Kad-19@*/aws_secure_deilvery_lab@*:ref:refs/heads/main
```
This keeps the condition scoped to the exact repo owner/name/branch while tolerating GitHub's current `sub` claim format.

---

## 5. `cdk diff` summary (captured from the live workflow run)

```
Stack: TrainingBasicStack

Resources to add (5):
  + AWS::S3::Bucket                         TrainingUploadsBucket
  + AWS::S3::BucketPolicy                   TrainingUploadsBucket/Policy
  + Custom::S3AutoDeleteObjects             TrainingUploadsBucket/AutoDeleteObjectsCustomResource
  + AWS::IAM::Role                          Custom::S3AutoDeleteObjectsCustomResourceProvider/Role
  + AWS::Lambda::Function                   Custom::S3AutoDeleteObjectsCustomResourceProvider/Handler

Resources to modify: 0
Resources to remove: 0

IAM/security-relevant changes:
  - New IAM role trusting lambda.amazonaws.com (sts:AssumeRole), used only by the CDK-generated
    S3 auto-delete-objects custom resource Lambda.
  - New bucket-scoped policy granting that Lambda's role s3:DeleteObject*, s3:GetBucket*,
    s3:List*, and s3:PutBucketPolicy on TrainingUploadsBucket only (not account-wide).
  - AWSLambdaBasicExecutionRole (AWS managed policy) attached to the custom resource role,
    for CloudWatch Logs write access only.
  - No public access, no cross-account access, no wildcard resource grants introduced.
```

---

## 6. Production change record

```
Purpose:
  Provision an S3 bucket (TrainingUploadsBucket) for the training stack, including CDK's
  standard auto-delete-on-destroy custom resource so the bucket can be cleanly torn down
  in non-production environments without manual object deletion.

Risk:
  Low. Purely additive (5 new resources, 0 modified, 0 removed). No existing resources are
  touched. The only IAM changes are scoped to a single new Lambda execution role, itself
  restricted to actions on this one bucket. No public access or cross-account access is
  introduced.

Expected diff:
  Exactly as captured in Section 5 — 1 S3 bucket, 1 bucket policy, 1 custom resource,
  1 IAM role, 1 Lambda function. Any diff showing additional/different resources on a
  future run should be treated as unexpected and investigated before approval.

Approval:
  PR review required before merge to main; deploy requires manual workflow_dispatch trigger
  by an authorized approver (no automatic deploy on push).

Smoke test:
  - Confirm bucket exists and is not publicly accessible:
    aws s3api get-bucket-policy-status --bucket <bucket-name>
  - Confirm the auto-delete custom resource Lambda executed successfully (CloudWatch Logs,
    no errors on stack create).
  - Confirm bucket encryption/versioning settings match the CDK stack definition.

Rollback:
  cdk destroy for this stack (safe due to auto-delete-objects custom resource — bucket will
  empty itself before deletion). Alternatively, redeploy the previous git commit's synthesized
  template. No data-loss risk since this is a net-new bucket with no pre-existing data.
```

---

## 7. Confirmation — no AWS access keys used

```
[x] No aws-access-key-id / aws-secret-access-key appear anywhere in the workflow file
[x] Authentication is exclusively via OIDC (configure-aws-credentials + role-to-assume)
[x] The only secret stored in GitHub is AWS_ROLE_ARN — a role identifier, not a credential
[x] No AWS access keys exist in repo or organization GitHub Secrets
[x] Local cdk diff runs (if performed) used temporary/SSO credentials, not long-lived IAM user keys
```

---
