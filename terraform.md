# Terraform

All LabLumen AWS infrastructure is defined as code in `lablumen-terraform`. The repository contains 12 modules — one per AWS service area — and all changes are applied through a gated CI/CD pipeline after an initial local bootstrap.

---

## Modules

| Module | AWS Resources Created |
|---|---|
| `vpc` | VPC (`10.0.0.0/16`), public / private / DB subnets across 2 AZs, NAT Gateway, Internet Gateway, VPC interface endpoints (SSM, Secrets Manager, SQS, ECR, Bedrock, Textract, CloudWatch Logs), S3 gateway endpoint |
| `eks` | EKS cluster `lablumen-eks` (v1.31), managed node group (`c7i-flex.large`), Karpenter IRSA and supporting resources, EKS Access Entries auth mode |
| `rds` | PostgreSQL 16.4 on `db.t4g.micro`, isolated DB subnet group, Secrets Manager-managed master password, single-AZ |
| `s3` | Reports bucket (private, KMS-encrypted, versioned, S3 EventBridge notifications enabled) and SAM artifacts bucket |
| `ecr` | 4 private image repositories (immutable tags, scan-on-push, KMS-encrypted) |
| `cognito` | User Pool `lablumen-users`, SPA app client (no secret), 3 user groups (`PATIENT`, `LAB_STAFF`, `LAB_ADMIN`) |
| `sqs` | Standard queue `lablumen-notifications` |
| `ses` | Domain identity for `rnld101.xyz`, Easy DKIM CNAME records created in Route 53 |
| `secretsmanager` | Empty secret shells for `lablumen/app/database-url` and `lablumen/app/grafana-admin` — values populated manually after apply |
| `ssm` | 15 non-sensitive config parameters under `/lablumen/config/*` (Cognito IDs, SQS URL, bucket names, etc.) |
| `iam` | GitHub OIDC provider, 5 CI/CD pipeline roles, IRSA roles for ESO, report-service, notification-service, AWS Load Balancer Controller, ExternalDNS |
| `lambda` | Lambda execution role and security group for the AI service |

`kubernetes.tf` (root, not a module) creates Kubernetes namespaces (`lablumen`, `lablumen-dev`, `external-secrets`) and IRSA-annotated ServiceAccounts after EKS is ready.

---

## State Management

Remote state lives in an S3 bucket using **S3-native locking** (Terraform 1.10+). No DynamoDB table is required.

The state bucket is created once via a standalone `bootstrap/` directory using local state. Its name is then hardcoded into `backend.tf`.

---

## Prerequisites

Terraform looks these up via data sources — it does not create them. They must exist before `terraform apply`:

1. A Route 53 hosted zone for `rnld101.xyz`
2. An ACM wildcard certificate (`*.rnld101.xyz`) in `us-east-1` with status `ISSUED`

---

## Initial Setup

```bash
# Step 1 — Create the S3 state bucket (run once, uses local state)
cd lablumen-terraform/bootstrap
terraform init && terraform apply
terraform output -raw state_bucket   # paste this value into ../backend.tf

# Step 2 — Apply all infrastructure (first time must be run locally)
cd ..
terraform init
terraform apply
```

The first `terraform apply` must run locally because the OIDC pipeline roles are created by Terraform itself — they do not exist yet, so the pipeline cannot authenticate.

After apply, two manual steps are required before the application can start:
1. Populate `lablumen/app/database-url` in Secrets Manager with the full PostgreSQL DSN
2. Populate `lablumen/app/grafana-admin` in Secrets Manager with Grafana credentials

---

## CI/CD Pipeline

All subsequent infrastructure changes go through a gated pipeline. No one applies manually after the initial setup.

| Stage | Trigger | IAM Role | What Happens |
|---|---|---|---|
| Scan | PR or push to `main` | — | Checkov IaC security scan; findings uploaded to GitHub Security tab |
| Plan | PR or push to `main` | `lablumen-tf-plan` (read-only) | `terraform fmt` → validate → plan; Infracost cost estimate posted as a PR comment |
| Apply | Push to `main` after human approval | `lablumen-tf-apply` (admin) | `terraform apply` using the exact plan artifact from the plan job |
| Destroy | Manual trigger only | `lablumen-tf-apply` | Guarded teardown — requires a typed confirmation string |

AWS credentials use GitHub OIDC — no static access keys. The `production` GitHub Environment provides the manual approval gate before apply runs.

---

## IAM Roles

| Role | Type | Assumed By |
|---|---|---|
| `lablumen-tf-plan` | CI/CD | Terraform repo, any branch — read-only plan access |
| `lablumen-tf-apply` | CI/CD | Terraform repo, `production` environment only — admin access |
| `lablumen-app-ci-ecr` | CI/CD | Any `lablumen/*` service repo — ECR push for backend images |
| `lablumen-frontend-build` | CI/CD | Frontend repo only — ECR push for the frontend image |
| `lablumen-ai-lambda-deploy` | CI/CD | AI service repo — SAM deploy permissions |
| `lablumen-eso` | IRSA | External Secrets Operator pods — Secrets Manager + SSM + KMS read |
| `lablumen-report-service` | IRSA | report-service pods — S3 access + Bedrock InvokeModel |
| `lablumen-notification-service` | IRSA | notification-service pods — SQS receive/delete + SES send |
| `lablumen-lbc` | IRSA | AWS Load Balancer Controller — manage ALBs from Ingress objects |
| `lablumen-external-dns` | IRSA | ExternalDNS — update Route 53 records from Ingress hostnames |

---

## Key Variables

| Variable | Value in `terraform.tfvars` | Notes |
|---|---|---|
| `domain_name` | `rnld101.xyz` | Used for SES identity, Route 53 records, ingress hostnames |
| `cluster_version` | `1.31` | EKS Kubernetes version |
| `node_instance_types` | `["c7i-flex.large"]` | Managed node group instance type |
| `db_instance_class` | `db.t4g.micro` | RDS instance sizing |
| `notifications_queue_name` | `lablumen-notifications` | SQS queue name |

---

## GitHub Configuration Required

| Setting | Where | Purpose |
|---|---|---|
| Variable `AWS_ACCOUNT_ID` | Org or repo | Constructs IAM role ARNs in workflow files |
| Environment `production` | `lablumen-terraform` repo settings | Required reviewers gate for `terraform apply` |
| Secret `INFRACOST_API_KEY` | `lablumen-terraform` repo secrets | PR cost estimate comments |
| Secret `BEDROCK_CROSS_ACCOUNT_ROLE_ARN` | `lablumen-terraform` repo secrets | Cross-account Bedrock IAM role ARN |
