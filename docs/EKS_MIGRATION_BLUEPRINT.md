# EKS / Terraform Migration Blueprint — LabLumen (Phase 3)

> **Document type:** Architectural planning, topology, and design-token reference.
> **Status:** Approved blueprint — no raw deployment scripts or complete `.tf` codebases are authored here.
> **Framing:** **Brownfield.** This document promotes and extends the infrastructure already scaffolded in
> `terraform/` and `k8s/`. It does not propose a clean-slate rebuild. Every "target" below is expressed as a
> delta against what already exists in the repository.
> **Author role:** Principal Solutions Architect / Lead Feature Engineer.

---

## 0. Overview & Guiding Principles

### 0.1 Where we are (verified single-EC2 state)

The platform has been validated end-to-end on a single EC2 instance using `docker-compose.yml`:

| Concern | Compose reality |
|---|---|
| Datastore | `pgvector/pgvector:pg16` container (local volume `pgdata`) |
| Cache / locks | `redis:7-alpine` container |
| Migrations | `migrate` one-shot (`alembic upgrade head`) gating the services |
| API | `appointment-service`, `report-service`, `notification-service` (FastAPI / uvicorn) |
| Edge | `frontend` nginx (port 80) serves the SPA and reverse-proxies `/api/v1/` to the services |
| AI | Textract + Bedrock currently reachable both via `serverless/ai-processing-pipeline` **and** an inline path inside `report-service` |

### 0.2 Where we are going (target production posture)

- **Compute:** Amazon EKS with a minimal bootstrap managed node group and **Karpenter** owning real capacity.
- **IaC:** Terraform remains the single source of truth for AWS primitives (`terraform/` modules).
- **Delivery:** **GitOps via ArgoCD**, evolving the existing `k8s/` app-of-apps tree.
- **Edge / routing:** **One shared public ALB → K-Gateway (Envoy) → `HTTPRoute` path matching.**
- **AI ingestion:** strictly the **S3-triggered AWS Lambda** (`serverless/ai-processing-pipeline`), VPC-attached
  to reach private RDS; the inline `report-service` AI path is retired.
- **Repos:** a **3-repo topology** — `lablumen-terraform` (infra), `lablumen-k8s` (GitOps), `lablumen-app`
  (consolidated `/backend` + `/frontend` + `/serverless`); architecture docs in `extras`.
- **Runtime config:** application pods read AWS Secrets Manager + SSM Parameter Store **directly via `boto3`/IRSA** —
  no Kubernetes Secrets, ConfigMaps, CSI driver, or External Secrets Operator.

### 0.3 The two hard boundaries (non-negotiable)

1. **Infra ⟂ App-deploy decoupling.** Terraform (infrastructure) and GitOps/CI (application releases) never share
   privileges. Terraform publishes **managed config** — sensitive values into **AWS Secrets Manager**
   (human-populated) and non-sensitive infra constants into **SSM Parameter Store** — which the application runtime
   reads directly via `boto3`/IRSA. The GitOps plane stays config-blind. No pipeline holds cross-boundary IAM.
2. **Lambda-native AI pipeline.** Document ingestion (Textract → Bedrock → pgvector) runs **only** inside the
   isolated Lambda, triggered by S3 `ObjectCreated`. EKS never performs OCR/embedding inline.

---

## 1. Foundational Layer — What Already Exists (Brownfield Inventory)

This is the substrate we build on. Do not reinvent these; extend them.

### 1.1 Terraform modules (`terraform/main.tf`)

```
module.network   -> terraform-aws-modules/vpc/aws   ~> 5.8
module.eks       -> terraform-aws-modules/eks/aws    ~> 20.24  (+ //modules/karpenter)
module.data      -> RDS Postgres 16 (pgvector)
module.storage   -> reports S3 bucket + module.ai_lambda (S3 ObjectCreated trigger)
module.messaging -> SQS notifications queue + SES sender
module.identity  -> Cognito user pool + IRSA (consumes module.eks.oidc_provider_arn)
```

| Module | Already provides | Gap to close in Phase 3 |
|---|---|---|
| `network` | Public `10.0.101-102.0/24`, private `10.0.1-2.0/24`, **single NAT**, ELB + `karpenter.sh/discovery` subnet tags | Add **isolated DB subnet tier**; add **VPC endpoints**; reserve room for K-Gateway/Lambda ENIs |
| `eks` | EKS `1.31`, static `t3.large` MNG (min1/max4/desired2), **`karpenter` submodule** (IAM role, instance profile, interruption SQS), node-SG discovery tag | Trim MNG to **bootstrap-only**; move real capacity to Karpenter `NodePool`/`EC2NodeClass` (GitOps) |
| `data` | RDS Postgres 16, `db.t3.medium`, 20 GB, in **private app subnets** | Relocate into the **isolated DB subnet group**; add SG ingress from Lambda SG |
| `storage` | Reports bucket + `module.ai_lambda` (python3.12, 512 MB, 60 s, S3 trigger wired) | Switch bucket to **default SSE-S3 (S3-managed keys)**; add Lambda **`VpcConfig`** + dedicated Lambda SG |
| `messaging` | SQS `lablumen-notifications` + SES | (no change) consumed by notification-service via IRSA |
| `identity` | Cognito + **IRSA foundation** (uses EKS OIDC ARN) | Add **per-service IRSA** granting scoped `secretsmanager:GetSecretValue` + `ssm:GetParameter(s)`; IRSA for LBC / Karpenter controller |

> **Design-token correction (storage):** the reports bucket must use **default server-side encryption with
> S3-managed keys (SSE-S3)**. Drop any explicit `aws:kms` server-side-encryption configuration. PHI confidentiality
> is satisfied by SSE-S3 at rest + full public-access block + TLS in transit.

### 1.2 Serverless AI pipeline (`serverless/ai-processing-pipeline/`)

The real ingestion pipeline already lives here: `src/handler.py` (S3 entry), `src/textract_ocr.py`,
`src/bedrock.py`, `src/chunking.py`, `src/db.py` (pgvector writes). `template.yaml` is the standalone SAM path;
in the Terraform-managed path, `terraform/modules/storage` owns the bucket and the `s3 -> lambda` notification.

> **Realignment:** `backend/report-service/` still carries inline `bedrock.py` / `textract.py` / `ingestion.py`.
> These are the temporary single-instance coupling. They are **removed** in Phase 3; report-service becomes a pure
> CRUD/query + presigned-URL service, and all OCR/embedding flows through the Lambda.

### 1.3 GitOps tree (`k8s/`) — already present

```
k8s/
├── root-app.yaml                         # ArgoCD app-of-apps root (path: k8s, recurse,
│                                          #   include: applications/*.yaml, platform-addons/*.yaml)
├── applications/                          # ArgoCD Application objects pointing at apps/* charts
│   ├── appointment-service.yaml
│   ├── report-service.yaml
│   ├── notification-service.yaml
│   └── redis.yaml
├── apps/                                  # Helm charts (one per workload)
│   ├── appointment-service/  (Chart, values, templates: deployment, service, hpa,
│   │                          serviceaccount, ingress)
│   ├── report-service/        (… + ingress)
│   ├── notification-service/  (no ingress — SQS consumer)
│   └── redis/
└── platform-addons/                       # Upstream addon Applications
    ├── argocd.yaml
    ├── aws-load-balancer-controller.yaml  # chart 1.8.2, clusterName: lablumen-eks
    ├── karpenter.yaml
    └── secrets-store-csi-driver.yaml       # ⚠ RETIRED by the direct-SDK model — removed in the config-model migration
```

**Routing today:** each service Helm chart renders a per-service **`Ingress`** (`apps/*/templates/ingress.yaml`),
materialized by the AWS Load Balancer Controller. **This is exactly what the K-Gateway model in §6 replaces.**

The GitOps source of record is currently `github.com/rnld101/lablumen.git`, path `k8s`. The repo split (§3) moves
this tree wholesale into **`lablumen-k8s`**.

---

## 2. Separation of Concerns — Infra vs. App Deployments

### 2.1 The boundary (three planes)

```
            TERRAFORM PLANE                          GITOPS / CI PLANE
   ┌──────────────────────────────┐        ┌──────────────────────────────────┐
   │ lablumen-terraform      │        │ lablumen-k8s         │
   │  • VPC / subnets / endpoints  │        │  • ArgoCD app-of-apps             │
   │  • EKS + Karpenter IAM        │        │  • Helm charts (apps/*)           │
   │  • RDS (pgvector)             │        │  • platform-addons/* (+k-gateway, │
   │  • S3 + ai_lambda + VpcConfig │        │    cluster-routing)               │
   │  • SQS / SES / Cognito        │        │  • Karpenter NodePool/EC2NodeClass│
   │  • IRSA roles                 │        │  • Gateway + HTTPRoutes           │
   └───────────────┬──────────────┘        └─────────────────┬────────────────┘
                   │ writes outputs                            │ reads
                   ▼                                           ▼
         ┌───────────────────────────────────────────────────────────┐
         │  AWS Secrets Manager (sensitive)  +  SSM Parameter Store   │
         │  (non-secret config)  +  GH Actions Secrets (CI → ECR)     │
         │  Terraform WRITES ─── app pods READ at runtime (boto3+IRSA)│
         └───────────────────────────────────────────────────────────┘
```

There are effectively **three** planes: Terraform *writes* managed config; the **GitOps/CI plane stays config-blind**
(declarative manifests only); the **application runtime reads** config via `boto3`+IRSA. No Terraform run reads the
GitOps repo; no Argo/CI pipeline holds Terraform state or cross-account IAM; the GitOps repo never reads secrets.

### 2.2 What flows across the boundary

Terraform **emits** these. Sensitive values land in **Secrets Manager** (empty containers created by Terraform,
values hand-populated by a human); non-sensitive infra constants are **published by Terraform into SSM Parameter
Store**. Applications read both **at runtime via `boto3`** — nothing flows through Helm/ConfigMaps.

| Value | Source (Terraform) | Sink | Consumer | Retrieval |
|---|---|---|---|---|
| `DATABASE_URL` (DSN incl. creds) | `module.data` + RDS master secret | Secrets Manager `lablumen/app/database-url` | services + `ai_lambda` | boto3 `get_secret_value` (TTL cache) |
| 3rd-party API keys / SMTP creds | human | Secrets Manager `lablumen/app/*` | owning service | boto3 `get_secret_value` (TTL cache) |
| Reports bucket name | `module.storage` | SSM `/lablumen/config/reports-bucket` | report-service, `ai_lambda` | boto3 `get_parameter` (process cache) |
| SQS queue URL | `module.messaging` | SSM `/lablumen/config/sqs-url` | appointment + notification | boto3 `get_parameter` (process cache) |
| Cognito pool / client IDs | `module.identity` | SSM `/lablumen/config/cognito-*` | services + SPA build | boto3 `get_parameter` / build-time |
| Bedrock model IDs | tfvars | SSM `/lablumen/config/bedrock-*` | report / ai | boto3 `get_parameter` |
| EKS OIDC provider ARN | `module.eks` | Terraform-internal | IRSA trust | n/a |
| ECR repo URIs | `aws_ecr_repository` | Helm `image.repository` + GH Actions | CI push + pods | git / Actions |

> **The one non-secret still in `lablumen-k8s`:** the ECR image URI (`image.repository`). It is part of the
> *deployment manifest*, not application *configuration*. Everything the app code reads comes from SSM / Secrets
> Manager, never from the chart.

### 2.3 Identifier classification — the runtime contract

Three lanes; the first two are read by the app at runtime via the SDK, and `lablumen-k8s` stays config-blind.

| Identifier | Lane | Sink | Retrieval | Cache |
|---|---|---|---|---|
| `DATABASE_URL`, API keys, SMTP creds | 🔒 Secret | AWS Secrets Manager | boto3 `get_secret_value` | TTL (default 900 s) → live rotation |
| Bucket names, SQS URL, Cognito IDs, model IDs | 📍 Config | SSM Parameter Store | boto3 `get_parameter(s)` | process lifetime (immutable constants) |
| ECR image URI | 📦 Deploy manifest | `lablumen-k8s` `values.yaml` | n/a (Helm renders it) | n/a |

> **Litmus test:** if leaking it forces a credential rotation → **Secret** (Secrets Manager, TTL cache). If it is
> stable infra topology the app needs to find its dependencies → **Config** (SSM Parameter Store, process cache). If
> it is only needed to *place* the container, not run its code → **Deploy manifest** (Helm values). Full lifecycle:
> `extras/docs/SECRETS_AND_CONFIG.md`.

---

## 3. Multi-Repository Decomposition Strategy

### 3.0 Canonical naming lock (3-repo topology — locked 2026-06-20)

The platform is **three** active repositories plus the docs vault. All earlier multi-repo draft names
(`lablumen-infra-terraform`, `lablumen-gitops-manifests`, and the per-service repos
`lablumen-appointment-service` / `lablumen-report-service` / `lablumen-notification-service` /
`lablumen-frontend` / `lablumen-ai-service`) are **retired** — their contents consolidate as below.

| Repo | Role | Holds |
|---|---|---|
| `lablumen-terraform` | AWS infrastructure plane | VPC, EKS, ECR, RDS, S3, SQS, SES, Cognito, Secrets Manager *containers*, SSM Parameter Store keys, IRSA |
| `lablumen-k8s` | GitOps declarative release plane | ArgoCD app-of-apps, `apps/*` Helm charts, `platform-addons/*` (incl. K-Gateway). **Config-blind** |
| `lablumen-app` | Consolidated application source | `/backend` (microservices), `/frontend` (React SPA), `/serverless` (Lambda pipeline) |
| `extras` | Documentation vault | blueprints, `CURRENT_STATUS.md`, `SECRETS_AND_CONFIG.md`, `MIGRATION_STATE_LEDGER.md` — no deployable code |

**Image & in-cluster identity map** (uniformly **singular**; all images built from `lablumen-app` via CI path filters):

| Workload | Source path (in `lablumen-app`) | ECR image repo | In-cluster identity (Service / Helm / Argo / HTTPRoute) |
|---|---|---|---|
| Appointment | `backend/appointment-service/` | `lablumen/appointment-service` | `appointment-service` |
| Reports | `backend/report-service/` | `lablumen/report-service` | `report-service` |
| Notification | `backend/notification-service/` | `lablumen/notification-service` | `notification-service` |
| Frontend | `frontend/` | `lablumen/frontend` | `frontend` |
| AI ingestion | `serverless/ai-processing-pipeline/` | — (zipped Lambda, no image) | `ai_lambda` (out-of-cluster) |

> **Naming is uniformly singular** (`report-service`, not `reports-service`) across the running k8s Service, the
> `lablumen-k8s/apps/report-service` chart, and the ECR image — matching the deployed tree. The earlier
> singular/plural split is retired.
>
> **Open drift (flagged):** `lablumen-terraform/main.tf` sets
> `module.storage.lambda_source_path = "../serverless/ai-processing-pipeline"` — valid only in a monorepo. With
> `lablumen-terraform` and `lablumen-app` as separate repos, the Lambda zip must be **built in `lablumen-app` CI and
> consumed by Terraform as an artifact** (published S3 object / build step). Resolve when wiring the AI bridge (ledger P6).

### 3.1 CI path-filter build matrix (`lablumen-app`)

One application repo, path-scoped pipelines: a push touching one service rebuilds only that service's image.
Illustrative GitHub Actions trigger (authored later in `lablumen-app`, not here):

```yaml
# .github/workflows/appointment-service.yml   (one workflow per workload)
on:
  push:
    paths: ['backend/appointment-service/**']
jobs:
  build-push:
    # docker build backend/appointment-service → push lablumen/appointment-service:<git-sha> → ECR
    # then PR-bump image.tag in lablumen-k8s/apps/appointment-service/values.yaml
```

| Path filter | Builds → pushes | Bumps tag in |
|---|---|---|
| `backend/appointment-service/**` | `lablumen/appointment-service` | `lablumen-k8s/apps/appointment-service` |
| `backend/report-service/**` | `lablumen/report-service` | `lablumen-k8s/apps/report-service` |
| `backend/notification-service/**` | `lablumen/notification-service` | `lablumen-k8s/apps/notification-service` |
| `frontend/**` | `lablumen/frontend` | `lablumen-k8s/apps/frontend` (P8) |
| `serverless/ai-processing-pipeline/**` | Lambda zip artifact | Terraform-consumed (no chart) |

### 3.2 Cross-cutting ownership decisions

- **Migrations (`alembic/`)** live in `lablumen-app/backend/appointment-service` (owns `0001_initial_schema.py`,
  `0002_seed_lab_tests.py`). The migration job is referenced (not duplicated) by the GitOps `apps/` chart as an
  ArgoCD **PreSync** hook.
- **Container registry:** one **ECR repo per containerized workload** (`lablumen/appointment-service`,
  `lablumen/report-service`, `lablumen/notification-service`, `lablumen/frontend`) — all built from `lablumen-app`.
  The AI pipeline ships as a **zipped Lambda — no ECR repo**.
- **Runtime config:** application code reads Secrets Manager + SSM Parameter Store directly via `boto3`
  (see `SECRETS_AND_CONFIG.md`); `lablumen-k8s` carries **no Secrets/ConfigMaps**.
- **GitOps source:** `root-app.yaml` `repoURL` points at `lablumen-k8s`.

---

## 4. Deliverable 1 — VPC & Network Layout

### 4.1 Three-tier topology

Extend today's two-tier (`network` module) into three tiers. **A single public Application Load Balancer faces
the internet and routes ALL traffic directly to the K-Gateway Envoy proxy pods running in the private app
subnets.** There is no per-service public exposure.

```
                          Internet
                             │
                     ┌───────▼────────┐
                     │ Internet Gateway│
                     └───────┬────────┘
   PUBLIC TIER (10.0.101.0/24, 10.0.102.0/24)
   ┌─────────────────────────┼──────────────────────────────┐
   │   Single shared public ALB   │   NAT Gateway (single)    │
   └─────────────────────────┼──────────────────────────────┘
                             │ (one target group → K-Gateway)
   PRIVATE APP TIER (10.0.1.0/24, 10.0.2.0/24)
   ┌─────────────────────────▼──────────────────────────────┐
   │  K-Gateway Envoy proxy pods  ◄── all north-south traffic │
   │  EKS worker nodes (Karpenter + bootstrap MNG)            │
   │  Service pods (appointment / report / notification)      │
   │  Lambda ENIs (ai_lambda VpcConfig)                       │
   └─────────────────────────┬──────────────────────────────┘
                             │ 5432 (SG-scoped)
   ISOLATED DB TIER (NEW — 10.0.201.0/24, 10.0.202.0/24)  no NAT, no IGW
   ┌─────────────────────────▼──────────────────────────────┐
   │  RDS PostgreSQL + pgvector  (db.t3.medium)              │
   └─────────────────────────────────────────────────────────┘
```

### 4.2 Subnet mapping guidelines (per AZ)

Today's defaults live in `terraform/variables.tf` (`private_subnets`, `public_subnets`). Add a **third list**
(`database_subnets`) and pass it through `module.network`.

| Tier | AZ `us-east-1a` | AZ `us-east-1b` | Routing | Purpose |
|---|---|---|---|---|
| Public | `10.0.101.0/24` | `10.0.102.0/24` | IGW default route | ALB, NAT GW |
| Private app | `10.0.1.0/24` | `10.0.2.0/24` | NAT default route + VPC endpoints | nodes, Karpenter, K-Gateway, service pods, Lambda ENIs |
| Isolated DB | `10.0.201.0/24` (new) | `10.0.202.0/24` (new) | **no default route** (intra) | RDS only |

**Required subnet tags** (already set on public/private by `modules/network/main.tf`, keep them):

```hcl
public  : "kubernetes.io/role/elb"          = 1
private : "kubernetes.io/role/internal-elb" = 1
private : "karpenter.sh/discovery"          = var.cluster_name
database: (no kubernetes/karpenter tags — RDS subnet group only)
```

> Sizing note: K-Gateway Envoy pods and Lambda ENIs both consume private-subnet IPs. A `/24` per AZ (251 usable)
> is adequate for MVP; if Karpenter scales aggressively, widen to `/23` or add secondary CIDRs before lock.

### 4.3 NAT strategy

Keep the **single NAT gateway** (`single_nat_gateway = true`) for cost. **VPC endpoints (below) absorb the
high-volume AWS-API egress** (S3, Textract, Bedrock, Secrets Manager, ECR), so NAT carries only incidental egress
and is not on the AI hot path.

### 4.4 VPC endpoints (new)

| Endpoint | Type | Why |
|---|---|---|
| `com.amazonaws.us-east-1.s3` | **Gateway** | Lambda + nodes pull report objects and ECR layers without NAT |
| `com.amazonaws.us-east-1.textract` | Interface | VPC-attached Lambda reaches Textract privately |
| `com.amazonaws.us-east-1.bedrock-runtime` | Interface | VPC-attached Lambda invokes Bedrock (Titan embed, Nova text) privately |
| `com.amazonaws.us-east-1.secretsmanager` | Interface | Pods (boto3) + Lambda resolve secrets (`DATABASE_URL`, keys) privately |
| `com.amazonaws.us-east-1.ssm` | Interface | Pods (boto3) read SSM Parameter Store config privately (off the NAT path) |
| `com.amazonaws.us-east-1.ecr.api` / `ecr.dkr` | Interface | Nodes pull service images privately |
| `com.amazonaws.us-east-1.logs` | Interface | CloudWatch Logs from in-VPC Lambda + nodes |
| `com.amazonaws.us-east-1.sqs` | Interface | notification-service + Karpenter interruption queue |

Interface endpoints live in the **private app subnets** with an endpoint SG allowing `443` from the VPC CIDR.

---

## 5. Deliverable 2 — EKS Compute Architecture with Karpenter

### 5.1 Node strategy

- **Bootstrap managed node group** (existing `eks_managed_node_groups.default`): trim from `desired 2 / max 4` to a
  minimal **`desired 1 / min 1 / max 2`** `t3.large` group. Its only job is to host the cluster-critical control
  pods that must exist *before* Karpenter is healthy: CoreDNS, the Karpenter controller itself, ArgoCD, AWS LB
  Controller, and the K-Gateway control plane.
- **All elastic workload capacity** (service pods, K-Gateway data plane at scale, burst) is provisioned by
  **Karpenter** via `NodePool` + `EC2NodeClass` CRDs delivered through GitOps (§6), not by Terraform.

### 5.2 Where Terraform ends and GitOps begins

```
Terraform (modules/eks):                       GitOps (platform-addons/karpenter + NodePools):
  • Karpenter controller IAM role               • Karpenter controller Helm release
  • Node IAM role + instance profile            • EC2NodeClass  (AMI family, subnet/SG discovery,
  • Interruption SQS queue                         instance profile ref)
  • Discovery tags on subnets + node SG         • NodePool      (instance families, capacity type=
  • Bootstrap MNG                                  spot/on-demand, limits, consolidation policy)
```

The discovery handshake already exists: `modules/network` tags private subnets with
`karpenter.sh/discovery = <cluster>`, and `modules/eks` tags the node SG the same. The `EC2NodeClass` selects
both by that tag — no ARNs hard-coded.

### 5.3 The Karpenter reconcile / EC2 Fleet loop

```
1. A Deployment scales up (HPA) or a new app syncs; pods land "Unschedulable"
   (insufficient cpu/mem, or nodeSelector/affinity unmet).
2. Karpenter watches the kube-scheduler's pending-pod stream via the API server.
3. For each unschedulable pod it computes a right-sized instance shape from the pod's
   resource requests + the matching NodePool constraints (families, arch, capacity type,
   zones, limits).
4. Karpenter calls the EC2 Fleet API (CreateFleet) — NOT an ASG — requesting that exact
   shape; with capacityType including "spot" it asks Fleet for the cheapest interruptible
   capacity, falling back to on-demand per the NodePool policy.
5. The new node boots from the EC2NodeClass AMI, joins via the bootstrap user-data, and
   registers. Karpenter binds the pending pods to it (it does not wait for kube-scheduler
   to rediscover capacity — it pre-binds).
6. Consolidation: when nodes are underutilized, Karpenter computes whether pods fit on
   fewer/cheaper nodes, drains, and terminates — continuously bin-packing.
7. Interruption: spot rebalance/termination notices land on the Terraform-provisioned
   interruption SQS queue; Karpenter drains the doomed node ahead of reclamation.
```

NodePool guidance (lives in GitOps): `c`/`m` families for services, `spot` first with on-demand fallback for
stateless API pods, on-demand-only taint for K-Gateway data plane if you want stable north-south capacity, and a
CPU limit ceiling to cap blast radius.

---

## 6. Deliverable 3 — GitOps & Helm Strategy (incl. K-Gateway routing)

### 6.1 ArgoCD model

ArgoCD is installed once, then the existing **app-of-apps** root (`k8s/root-app.yaml`) takes over. Two valid
bootstrap options:

- **Terraform Helm provider** installs ArgoCD as part of the infra bootstrap (keeps "cluster exists ⇒ Argo
  exists" atomic), **or**
- a one-time `kubectl apply -f root-app.yaml` seed after a manual ArgoCD install.

Either way, `root-app` then continuously reconciles everything under `k8s/` (now `lablumen-k8s`).

### 6.2 Target repository layout (`lablumen-k8s`)

```
gitops/
├── root-app.yaml                       # include glob UPDATED (see §6.5)
├── applications/                       # ArgoCD Applications (one per chart / addon group)
│   ├── appointment-service.yaml
│   ├── report-service.yaml
│   ├── notification-service.yaml
│   └── redis.yaml
├── apps/                               # Service Helm charts
│   ├── appointment-service/   (ingress.yaml REMOVED)
│   ├── report-service/        (ingress.yaml REMOVED)
│   ├── notification-service/
│   └── redis/
└── platform-addons/
    ├── argocd.yaml
    ├── aws-load-balancer-controller.yaml   # now manages ONE shared ALB → K-Gateway
    ├── karpenter.yaml                       # + NodePool / EC2NodeClass
    ├── k-gateway/              # NEW — Helm charts: K-Gateway control plane + Envoy data plane
    └── cluster-routing/        # NEW — Gateway API CRDs + Gateway + HTTPRoute resources
                                #   (NO secrets-store-csi-driver — apps read SSM/Secrets Manager via boto3)
```

### 6.3 The K-Gateway routing topology (replaces per-service Ingress)

```
            Internet
               │
        ┌──────▼───────┐   AWS Load Balancer Controller manages EXACTLY ONE
        │ Shared public │  shared ALB. Its single target group → K-Gateway Envoy.
        │     ALB       │  (No per-service ALB. The old apps/*/ingress.yaml are gone.)
        └──────┬───────┘
               │  one target group
        ┌──────▼─────────────────────────────┐
        │  K-Gateway (Envoy) data plane pods   │  ← does host/path matching itself
        │  bound to the Gateway "lablumen-gw"  │
        └───┬───────────────┬─────────────┬────┘
   HTTPRoute│       HTTPRoute│     HTTPRoute│
  /api/v1/  │  /api/v1/reports│   /api/v1/  │ (catch-all → appointment)
  appointmts│   report-service│   …         │
        ┌───▼───┐       ┌─────▼────┐   ┌────▼──────────┐
        │appt-svc│       │report-svc│   │notification…  │
        └────────┘       └──────────┘   └───────────────┘
```

**Division of responsibility (state this explicitly):**

- **AWS Load Balancer Controller** is responsible for **one shared public ALB only**, whose single backend is the
  K-Gateway Envoy service. It no longer renders an ALB per service.
- **K-Gateway** owns **L7 path matching**. It binds to a single `Gateway` (`lablumen-gw`) and selects backends via
  Gateway API **`HTTPRoute`** resources:
  - `/api/v1/reports` → `report-service`
  - `/api/v1/appointments`, `/api/v1/patients`, `/api/v1/lab-tests` (i.e. `/api/v1/`) → `appointment-service`
  - notification-service has no public route (SQS consumer).

`HTTPRoute` shape (lives in `platform-addons/cluster-routing/`, illustrative structure — not a deployment script):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: report-routes
spec:
  parentRefs:
    - name: lablumen-gw            # the single cluster Gateway bound to K-Gateway
  rules:
    - matches:
        - path: { type: PathPrefix, value: /api/v1/reports }
      backendRefs:
        - name: report-service
          port: 8000
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: appointment-routes
spec:
  parentRefs:
    - name: lablumen-gw
  rules:
    - matches:
        - path: { type: PathPrefix, value: /api/v1 }   # broad prefix, evaluated after /reports
      backendRefs:
        - name: appointment-service
          port: 8000
```

> More-specific prefixes (`/api/v1/reports`) must out-rank the broad `/api/v1` per Gateway API precedence — keep
> them in separate `HTTPRoute`s so K-Gateway resolves longest-prefix-wins.

### 6.4 Runtime config (direct AWS SDK — no CSI/ESO, no K8s Secrets/ConfigMaps)

Application config does **not** flow through Kubernetes. Each pod, authenticated by its **IRSA** service account,
reads **AWS Secrets Manager** (sensitive — `boto3 get_secret_value`, TTL-cached for live rotation) and **SSM
Parameter Store** (non-sensitive infra constants — `boto3 get_parameter`, process-cached) **at startup**. There is
no `SecretProviderClass`, no External Secrets `ExternalSecret`, no CSI-mounted volume, and no `Secret`/`ConfigMap`
in the charts. Bootstrap implements retry-with-backoff and **fails closed** if mandatory config is unavailable.
Full lifecycle: `extras/docs/SECRETS_AND_CONFIG.md`.

### 6.5 Sync waves & root-app glob

- Update `root-app.yaml` `directory.include` from
  `"{applications/*.yaml,platform-addons/*.yaml}"` to also pick up the new routing tree, e.g.
  `"{applications/*.yaml,platform-addons/*.yaml,platform-addons/cluster-routing/*.yaml}"` (or register
  `k-gateway` / `cluster-routing` as their own Applications under `applications/`).
- **Sync-wave ordering:** Gateway API CRDs (wave 0, Terraform-bootstrapped) → platform addons incl. K-Gateway +
  LBC + Karpenter (wave 1) → `Gateway` + `HTTPRoute` routing (wave 2) → service apps (wave 3). Migrations run as
  the appointment-service **PreSync** hook.

### 6.6 Image-tag promotion

CI builds and pushes immutable image tags (git SHA) to ECR, then opens a PR against `lablumen-k8s`
bumping the chart `image.tag`. ArgoCD detects the commit and syncs. Promotion = a git commit, never a direct
`kubectl`/`helm` apply.

---

## 7. Deliverable 4 — Serverless AI Pipeline Realignment

### 7.1 Target flow

```
report-service (presign) ──► client uploads PDF ──► S3 reports bucket
                                                        │ s3:ObjectCreated:*
                                                        ▼
                                          ai_lambda (VPC-attached)
                          ┌─────────────────────────────┼─────────────────────────────┐
                          ▼ Textract (VPC endpoint)      ▼ Bedrock (VPC endpoint)        ▼ Secrets Mgr (endpoint)
                    OCR text                       embeddings (Titan) +            DATABASE_URL
                          │                         summary (Nova)                       │
                          └──────────────► chunk + write vectors ─────────────────► RDS pgvector (5432, SG-scoped)
```

`report-service` keeps only: presigned upload/download, report metadata CRUD, and pgvector **query** for the
chat/summary read path. Its inline `bedrock.py` / `textract.py` / `ingestion.py` are deleted.

### 7.2 The network bridge

| Element | Spec |
|---|---|
| Lambda placement | **Private app subnets** (`10.0.1.0/24`, `10.0.2.0/24`) via `VpcConfig` |
| Lambda SG (new) | `lablumen-ai-lambda-sg` — no inbound; egress `443` (AWS APIs via endpoints) + `5432` to RDS SG |
| RDS SG rule (new) | Ingress `5432` **from `lablumen-ai-lambda-sg`** (and from the EKS node SG for services) |
| AWS API reachability | Textract / Bedrock / Secrets Manager / S3 reached via the **VPC endpoints** in §4.4 (a VPC-attached Lambda loses default internet egress, so endpoints are mandatory) |
| `DATABASE_URL` | resolved from **Secrets Manager** at runtime via `boto3` (TTL-cached); never baked into the zip |

### 7.3 `VpcConfig` block shape to add to `module.ai_lambda`

Illustrative shape (design token, **not** a complete `.tf` to apply in this step) — `module.storage` would pass
through subnet/SG inputs sourced from `module.network` and a new Lambda SG:

```hcl
# terraform/modules/storage/main.tf  (module.ai_lambda)
  vpc_subnet_ids         = var.private_subnet_ids          # from module.network.private_subnets
  vpc_security_group_ids = [var.ai_lambda_security_group_id]
  attach_network_policy  = true                            # adds AWSLambdaVPCAccessExecutionRole (ENI mgmt)

  environment_variables = {
    BEDROCK_EMBED_MODEL_ID = "amazon.titan-embed-text-v1"
    # us-east-1 on-demand only — org SCP p-rn6vr8ok denies the Nova 2 inference-profile regions.
    BEDROCK_TEXT_MODEL_ID  = "amazon.nova-lite-v1:0"
    # DATABASE_URL injected from Secrets Manager at deploy time
  }
```

Corresponding SG sketch (design token):

```hcl
resource "aws_security_group" "ai_lambda" {
  name   = "lablumen-ai-lambda-sg"
  vpc_id = var.vpc_id
  egress { from_port = 443  to_port = 443  protocol = "tcp" cidr_blocks = [var.vpc_cidr] } # endpoints
  egress { from_port = 5432 to_port = 5432 protocol = "tcp" security_groups = [var.rds_sg_id] }
}

resource "aws_security_group_rule" "rds_from_lambda" {
  type                     = "ingress"
  from_port                = 5432
  to_port                  = 5432
  protocol                 = "tcp"
  security_group_id        = var.rds_sg_id
  source_security_group_id = aws_security_group.ai_lambda.id
}
```

> The existing `aws_lambda_permission.allow_s3` + `aws_s3_bucket_notification.reports` in `modules/storage`
> already wire the S3 trigger — only the VPC attachment + SG bridge are additive.

---

## 8. Deliverable 5 — Step-by-Step Execution Phases

Each phase is **build → test → lock** (do not start the next phase until the current one's lock criteria pass).

| Phase | Build | Test | Lock criteria |
|---|---|---|---|
| **P0 — Foundations** | Terraform remote state (S3 + DynamoDB lock); Secrets Manager containers; **SSM Parameter Store baseline paths**; ECR repos | `terraform plan` clean; secrets/params readable by intended principals only | State backend + secret/param scaffolding immutable & reviewed |
| **P1 — Network** | Add `database_subnets` tier; add VPC endpoints (§4.4); keep single NAT | Endpoint DNS resolves in-VPC; routing tables correct per tier | Subnet CIDRs + endpoints locked; tags verified |
| **P2 — Data** | Move RDS into the **isolated DB subnet group**; SG baseline | Connectivity only from app tier; no public route | RDS endpoint published to Secrets Manager; tier isolation confirmed |
| **P3 — EKS core** | EKS + **bootstrap-only** MNG + Karpenter controller IAM/instance-profile/interruption queue | Cluster reachable; bootstrap nodes Ready; discovery tags present | Control-plane + bootstrap capacity locked |
| **P4 — Gateway CRDs + add-ons** | **Bootstrap the standard Kubernetes Gateway API CRDs via the Terraform Helm provider FIRST**, then install ArgoCD + sync `platform-addons` (LBC, Karpenter, **k-gateway**; no CSI) | `kubectl get crd` shows `gateway.networking.k8s.io`; ArgoCD healthy; K-Gateway control plane up | Gateway API CRDs exist **before** any `HTTPRoute` syncs; addons green |
| **P5 — Karpenter live** | Apply `NodePool`/`EC2NodeClass` via GitOps; scale-test; then retire static nodes | Pending pods trigger CreateFleet; consolidation works; spot interruption drains cleanly | Real capacity on Karpenter; bootstrap MNG at min |
| **P6 — AI bridge** | Add `ai_lambda` `VpcConfig` + Lambda SG + RDS ingress rule; remove report-service inline AI path | Upload → S3 event → Lambda → Textract/Bedrock via endpoints → pgvector write; report-service query reads vectors | Ingestion 100% via Lambda; inline path deleted |
| **P7 — Repo split + delivery** | Consolidate into **3 repos** (`lablumen-terraform` / `lablumen-k8s` / `lablumen-app`); CI **path filters** in `lablumen-app` build+push per-service images to ECR; repoint `root-app` at `lablumen-k8s` | Each path-filtered workflow builds/tests its workload; ArgoCD tracks `lablumen-k8s` | Monorepo frozen; per-workload pipelines green |
| **P8 — Routing cutover** | Deploy the frontend (`lablumen-app/frontend`) and the **shared-ALB → K-Gateway** wiring and the per-service **`HTTPRoute`** manifests (replacing the old per-service ALB Ingress) | `/api/v1/reports` and `/api/v1/` resolve through the single ALB → K-Gateway → correct service | One public ALB only; HTTPRoutes authoritative; service Ingress removed |
| **P9 — Decommission** | Cut DNS to the new ALB; drain compose; archive EC2 stack | Full E2E on EKS; no traffic to compose | `docker-compose.yml` retired; EKS is sole runtime |

---

## 9. Appendix

### 9.1 Terraform design tokens

| Token | Convention |
|---|---|
| Naming | `lablumen-<resource>` (e.g. `lablumen-eks`, `lablumen-pg`, `lablumen-ai-processing`, `lablumen-ai-lambda-sg`) |
| Cluster name | `local.cluster_name = "${var.project}-eks"` (already in `main.tf`) |
| Tags | `{ Project = "lablumen", ManagedBy = "terraform" }` merged via provider `default_tags` |
| Module boundary | one module per AWS domain (`network`/`eks`/`data`/`storage`/`messaging`/`identity`); modules expose typed outputs, never reach across each other except through `main.tf` wiring |
| Discovery | subnet/SG `karpenter.sh/discovery = <cluster>`, `kubernetes.io/role/*-elb` — tag-based, no hard-coded ARNs |
| Secrets / config | sensitive → Secrets Manager; non-sensitive → SSM Parameter Store; both read at runtime via `boto3`/IRSA; never in `*.tfvars` or git |
| Region pin | `us-east-1` (org SCP constraint — see Bedrock model note in `docker-compose.yml`) |

### 9.2 "What changes per existing module" — quick reference

| Module / dir | Change |
|---|---|
| `modules/network` | + `database_subnets`; + VPC endpoints (S3 gateway, Textract/Bedrock/SecretsMgr/ECR/Logs/SQS interface) |
| `modules/eks` | bootstrap MNG → `min1/desired1/max2`; Karpenter `NodePool`/`EC2NodeClass` move to GitOps |
| `modules/data` | RDS → isolated DB subnet group; + ingress from Lambda SG |
| `modules/storage` | bucket → **SSE-S3**; `ai_lambda` → `VpcConfig` + `lablumen-ai-lambda-sg` |
| `secrets.tf` / `ssm.tf` (root, new) | Secrets Manager *containers* (no `_version`) + SSM Parameter Store config keys (`/lablumen/config/*`) |
| `modules/identity` | + **per-service IRSA** (scoped `secretsmanager:GetSecretValue` + `ssm:GetParameter(s)`); IRSA for LBC / Karpenter controller |
| root (`lablumen-terraform`) | + Helm-provider bootstrap of **Gateway API CRDs** (P4) |
| `k8s/` → `lablumen-k8s` | + `platform-addons/k-gateway`, + `platform-addons/cluster-routing`; **remove** `apps/*/templates/ingress.yaml`; **remove** `secrets-store-csi-driver`; charts carry **no Secrets/ConfigMaps**; update `root-app.yaml` include glob + `repoURL` |
| `lablumen-app/backend/report-service` | remove inline `bedrock.py`/`textract.py`/`ingestion.py` (AI flows through the Lambda) |
| `docker-compose.yml` | retired at P9 |

### 9.3 End-state topology (single picture)

```
Internet ─► IGW ─► [Public tier] Single ALB ──► [Private app tier] K-Gateway Envoy
                                                      │  HTTPRoute path-match
                                       ┌──────────────┼───────────────┐
                                   appointment     report          notification (SQS only)
                                       │               │                │
                                   Karpenter-managed nodes (spot+on-demand), bootstrap MNG (min)
                                       │
S3 reports ─(ObjectCreated)─► ai_lambda (VPC) ─endpoints─► Textract/Bedrock/SecretsMgr
                                       │
                              [Isolated DB tier] RDS Postgres + pgvector  (5432, SG-scoped)

Terraform ──writes──► Secrets Manager + SSM Parameter Store ◄──reads at runtime (boto3+IRSA)── app pods
                       (GitOps plane is config-blind; CI uses GH Actions Secrets only to push images to ECR)
```

---

*End of blueprint. This document defines topology, boundaries, and Terraform design tokens for Phase 3. Raw
deployment workflows, complete `.tf` files, and Helm chart bodies are produced in their respective repositories in
later phases, pathing toward the GitHub Actions integration described in §2 and §6.6.*
