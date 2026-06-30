# LabLumen

LabLumen is a cloud-native laboratory management platform built on AWS. Patients book lab tests, receive AI-analyzed reports, and chat with an AI assistant about their results. Lab staff manage the operations queue, upload reports, and track appointment statuses through the same application.

![LabLumen Login](images/lablumen_login.png)

---

## Platform Architecture

The platform is composed of four backend services, one React frontend, and one serverless AI pipeline — all running on AWS in `us-east-1`. Authentication is handled entirely by Amazon Cognito; there is no separate auth service.

![Application Architecture](images/app_architecture_diagram.png)

| Service | Role |
|---|---|
| `appointment-service` | Lab test catalog, patient profiles, appointment booking, distributed slot locking |
| `report-service` | PDF report upload and storage, presigned viewing URLs, AI chat (RAG) |
| `notification-service` | SQS consumer — sends appointment confirmation emails via SES |
| `ai-service` | AWS Lambda — OCR, AI summarization, and vector embedding for every uploaded report |
| `frontend` | React 18 SPA served by nginx in EKS; nginx also reverse-proxies API calls to backend services |

---

## AWS Infrastructure

All infrastructure is provisioned with Terraform across a custom VPC in `us-east-1`. Application services run on an EKS cluster. The AI pipeline runs serverless via AWS Lambda.

![AWS Infrastructure Architecture](images/aws_architecture_diagram.png)

---

## Application

### Patient Portal

Patients log in, book lab tests, track upcoming appointments, view their PDF reports alongside an AI-generated plain-English summary, and chat with an AI assistant about their specific results.

![Patient Appointments](images/lablumen_appointment_service.png)

![AI Report Analysis](images/lablumen_ai_service.png)

### Staff Portal

Lab staff see every patient appointment in an operations queue, update statuses, and upload PDF lab reports directly from the dashboard.

![Staff Operations Queue](images/lablumen_staff.png)

---

## Repositories

| Repository | Description |
|---|---|
| `lablumen-appointment-service` | FastAPI service — appointments, lab test catalog, patient profiles |
| `lablumen-report-service` | FastAPI service — report upload, S3 storage, presigned URLs, AI chat |
| `lablumen-notification-service` | FastAPI service — SQS consumer, SES email delivery |
| `lablumen-ai-service` | AWS Lambda — OCR, AI summarization, vector embedding (SAM-deployed) |
| `lablumen-frontend` | React 18 + Vite SPA, served by nginx in EKS |
| `lablumen-terraform` | All AWS infrastructure as code — 12 Terraform modules |
| `lablumen-k8s` | GitOps repo — ArgoCD App-of-Apps, Helm values, cluster configuration |
| `lablumen-shared` | Reusable GitHub Actions workflows called by all service repositories |

---

## Technology

| Area | Technology |
|---|---|
| Backend services | Python 3.12, FastAPI, SQLAlchemy (async), Alembic, Pydantic v2 |
| Frontend | React 18, TypeScript, Vite, Tailwind CSS, TanStack Query |
| Database | Amazon RDS PostgreSQL 16.4 with pgvector extension |
| AI | Amazon Bedrock (Nova Lite for summaries, Titan Embed for vectors), AWS Textract for OCR |
| Auth | Amazon Cognito — SRP flow, JWT, group-based RBAC (`PATIENT`, `LAB_STAFF`, `LAB_ADMIN`) |
| Messaging | Amazon SQS + Amazon SES |
| Container platform | Amazon EKS (Kubernetes 1.31), Karpenter, ArgoCD, Helm |
| Infrastructure | Terraform 1.10+, 12 modules |
| CI/CD | GitHub Actions, OIDC auth — no static AWS credentials anywhere |
| Container registry | Amazon ECR (immutable tags, KMS-encrypted) |

---

## Documentation

- [Kubernetes](kubernetes.md) — EKS cluster, namespaces, workloads, platform add-ons, GitOps
- [Terraform](terraform.md) — Infrastructure modules, bootstrap process, CI/CD pipeline
- [GitHub Actions](github_actions.md) — Shared workflows, security gates, OIDC auth, build-once promote
- [AWS](aws.md) — Every AWS service used and its role in the platform

---

## Deployment

See the [Deployment Runbook](runbook.md) for the complete step-by-step guide:

1. GitHub organization setup — secrets, variables, environments
2. AWS prerequisites — domain, ACM certificate
3. Terraform state bucket bootstrap
4. Terraform apply — full infrastructure
5. Secrets Manager population — database DSN and credentials
6. ArgoCD cluster bootstrap
7. CI/CD first run and production validation

---

## GitHub Organization Setup

Configure the following before any pipeline can run. Most settings belong at the **organization level** so all repositories inherit them.

### Organization Variables

| Variable | Description |
|---|---|
| `AWS_ACCOUNT_ID` | 12-digit AWS account ID — used to construct IAM role ARNs in every workflow |
| `SONAR_ORGANIZATION` | SonarCloud organization key |

### Organization Secrets

| Secret | Description |
|---|---|
| `K8S_REPO_PAT` | Personal access token with `repo` write access to `lablumen-k8s` (GitOps write-back) |
| `SONAR_TOKEN` | SonarCloud API token |
| `SNYK_TOKEN` | Snyk API token for dependency vulnerability scanning |

### Terraform Repository — Additional Secrets

Set these on the `lablumen-terraform` repository only (not org-wide):

| Secret | Description |
|---|---|
| `INFRACOST_API_KEY` | Infracost API key — cost estimates are posted as PR comments |
| `BEDROCK_CROSS_ACCOUNT_ROLE_ARN` | IAM role ARN in the Bedrock-enabled AWS account |

### Terraform Repository — GitHub Environment

Create a GitHub Environment named **`production`** on `lablumen-terraform` with required reviewers. This is the manual approval gate that blocks `terraform apply` until a human approves the plan.
