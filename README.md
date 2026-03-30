# AWS Automated Baseline IAM Config
 
This hands-on lab uses AWS CloudFormation to automatically configure AWS Identity and Access Management (IAM) groups, policies, and roles for cross-account access — following AWS security best practices.
 
---
 
## Architecture Overview
 
The baseline is delivered as a **CloudFormation nested-stack** pattern. A single master stack orchestrates four child stacks, each responsible for one layer of the IAM configuration.
 
```
master.yaml
├── account-password-policy.yaml   (Lambda custom resource)
├── iam-managed-policies.yaml      (customer-managed policies)
├── iam-groups.yaml                (IAM groups)
└── iam-cross-account-roles.yaml   (cross-account roles)
```
 
---
 
## What Gets Deployed
 
### Account Password Policy
| Setting | Value |
|---|---|
| Minimum length | 14 characters |
| Character requirements | Uppercase, lowercase, numbers, symbols |
| Maximum password age | 90 days |
| Password reuse prevention | Last 24 passwords |
| Allow user self-service | Yes |
 
### Customer-Managed Policies
 
| Policy | Purpose |
|---|---|
| `EnforceMFA` | Denies all API calls unless the session has an active MFA token. Users can still enrol their MFA device and change a temporary password before MFA is active. |
| `DenyRootActions` | Denies high-risk IAM and billing mutations when called by the root principal. |
| `DenyUnencryptedS3Uploads` | Denies `s3:PutObject` calls that omit a server-side encryption header. |
| `RequireSecureTransport` | Denies any API call made over plain HTTP. |
 
### IAM Groups
 
| Group | AWS Managed Policy | Extra Restrictions |
|---|---|---|
| `Administrators` | AdministratorAccess | EnforceMFA + DenyRootActions |
| `Developers` | PowerUserAccess | EnforceMFA + DenyIAMEscalation inline policy |
| `ReadOnly` | ReadOnlyAccess | EnforceMFA |
| `SecurityAuditors` | SecurityAudit + SecurityHub/GuardDuty/Config read | EnforceMFA + extended security-service read |
| `BillingAdmins` | Billing job function | EnforceMFA + Cost Explorer access |
 
### Cross-Account Roles
 
All roles share these security controls:
- **ExternalId** required in `sts:AssumeRole` call (confused-deputy protection)
- **MFA required** — `aws:MultiFactorAuthPresent: true`
- **MFA token age** — must be less than 1 hour old
- **Maximum session duration** — 1 hour (configurable)
 
| Role | Access Level | Intended Use |
|---|---|---|
| `CrossAccountAdminRole` | AdministratorAccess | Break-glass / emergency only |
| `CrossAccountDeveloperRole` | PowerUser (no IAM write) | Day-to-day engineering work |
| `CrossAccountReadOnlyRole` | ReadOnlyAccess | Audits, monitoring, support |
 
---
 
## Project Structure
 
```
.
├── cloudformation/
│   ├── master.yaml                        # Root stack — deploy this one
│   └── templates/
│       ├── account-password-policy.yaml   # Password policy (Lambda custom resource)
│       ├── iam-managed-policies.yaml      # Customer-managed policies
│       ├── iam-groups.yaml                # IAM groups
│       └── iam-cross-account-roles.yaml   # Cross-account IAM roles
└── parameters/
    └── master-params-example.json         # Example parameter file
```
 
---
 
## Prerequisites
 
- AWS CLI configured with credentials that have `iam:*` and `cloudformation:*` permissions.
- An S3 bucket in the same region as your target account to host the nested templates.
 
---
 
## Deployment
 
### 1. Upload nested templates to S3
 
```bash
BUCKET=my-cfn-templates-bucket
PREFIX=cloudformation/templates
 
aws s3 sync cloudformation/templates/ s3://${BUCKET}/${PREFIX}/
```
 
### 2. Copy and edit the parameter file
 
```bash
cp parameters/master-params-example.json parameters/master-params.json
# Edit master-params.json and set:
#   TemplateBucketName  → your S3 bucket name
#   TrustedAccountId    → the 12-digit account ID allowed to assume cross-account roles
#   ExternalId          → a random shared secret (min 8 characters)
```
 
### 3. Deploy the master stack
 
```bash
aws cloudformation deploy \
  --stack-name iam-baseline \
  --template-file cloudformation/master.yaml \
  --parameter-overrides file://parameters/master-params.json \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```
 
### 4. Verify deployment
 
```bash
aws cloudformation describe-stacks \
  --stack-name iam-baseline \
  --query 'Stacks[0].StackStatus'
```
 
---
 
## Security Best Practices Applied
 
| Practice | Implementation |
|---|---|
| Least privilege | Each group and role scoped to the minimum required access |
| MFA enforcement | `EnforceMFA` policy on every human-user group and role |
| No privilege escalation | `DenyIAMEscalation` inline policy on Developer group/role |
| Confused-deputy protection | `ExternalId` condition on all cross-account role trust policies |
| Encryption in transit | `RequireSecureTransport` policy denies HTTP API calls |
| Encryption at rest | `DenyUnencryptedS3Uploads` policy enforces SSE on all S3 puts |
| Short-lived sessions | Cross-account role sessions capped at 1 hour |
| Strong password policy | 14-char minimum, complexity requirements, 90-day rotation |
 
---
 
## License
 
MIT — see [LICENSE](LICENSE).