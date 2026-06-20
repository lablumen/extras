# LabLumen EKS Migration — State Ledger (P0 → P9)

> **Purpose:** Single persistent source of truth for migration progress, resistant to context loss / restarts.
> Each phase declares **Build Inputs**, **Resource Vectors** (the concrete files/AWS resources), and **binary
> Lock / Verification Criteria**. A phase is `LOCKED` only when **every** box is checked. Do not start phase N+1
> until phase N is `LOCKED`. Companion docs: `EKS_MIGRATION_BLUEPRINT.md` (topology), `SECRETS_AND_CONFIG.md`
> (runtime config). Last updated: 2026-06-20 (P0 ssm.tf + iam.tf authored; P1 network module authored; fmt+validate clean).

## Canonical context (locked)

- **Repos (3 + docs):** `lablumen-terraform` (infra), `lablumen-k8s` (GitOps, config-blind), `lablumen-app`
  (`/backend` + `/frontend` + `/serverless`), `extras` (docs).
- **Config model:** **External Secrets Operator (ESO)** — ESO controller authenticates via a cluster-level IRSA role,
  syncs Secrets Manager + SSM Parameter Store into ephemeral K8s Secrets, pods consume as env vars. No Direct SDK
  config calls in app code. No hardcoded Secrets/ConfigMaps in git. Auth via ESO IRSA only; per-service IRSA for
  operational AWS calls (S3 / Bedrock / SQS / SES) only.
- **Region:** `us-east-1` (SCP `p-rn6vr8ok`). **Bedrock text model:** `amazon.nova-lite-v1:0`.
- **Naming:** uniformly singular (`report-service`). ECR `lablumen/<service>`, built from `lablumen-app` via CI path filters.

## Status legend

`[ ]` open · `[x]` done · **NOT STARTED** / **IN PROGRESS** / **LOCKED**

## Progress snapshot (2026-06-20)

| Phase | State | Note |
|---|---|---|
| P0 Foundations | **IN PROGRESS** | Bootstrap state, ECR, SM containers, SSM params (`ssm.tf`) + ESO + operational IRSA roles (`iam.tf`) all **authored** (not applied). Awaiting bootstrap apply + remote-state migration. |
| P1 Network | **IN PROGRESS** | 3-AZ three-tier VPC + 9 VPC endpoints **authored** (not applied). Awaiting P0 LOCKED before apply. |
| P2 Data | NOT STARTED | |
| P3 EKS core | NOT STARTED | |
| P4–P9 | NOT STARTED | See blueprint §8 for summaries. |

---

## Phase 0 — Foundations  **[IN PROGRESS]**

**Build Inputs:** AWS account `130290476321`, `us-east-1`; existing `lablumen-terraform` modules; canonical naming lock.

**Resource Vectors:**
- `lablumen-terraform/bootstrap/` — `aws_s3_bucket.tfstate` (`lablumen-tfstate`, versioned, AES256, public-access-blocked, `prevent_destroy`) + `aws_dynamodb_table.tflock` (`lablumen-tflock`).
- `lablumen-terraform/versions.tf` — `backend "s3"` block enabled.
- `lablumen-terraform/ecr.tf` — `aws_ecr_repository.services` for `lablumen/{appointment-service,report-service,notification-service,frontend}` (IMMUTABLE tags, scan-on-push, AES256, keep-last-20).
- `lablumen-terraform/secrets.tf` — `aws_secretsmanager_secret` containers (`lablumen/app/database-url`, …) — **no `_version`**.
- `lablumen-terraform/ssm.tf` — **(TO AUTHOR)** `aws_ssm_parameter` for `/lablumen/config/*` baseline keys.
- `lablumen-terraform/iam.tf` (or `identity.tf`) — **(TO AUTHOR)** `aws_iam_role.eso`: trust policy scoped to
  `system:serviceaccount:external-secrets:external-secrets`; inline policy grants
  `secretsmanager:GetSecretValue` on `lablumen/app/*` and `ssm:GetParameter` / `ssm:GetParametersByPath` on
  `/lablumen/config/*` only (no `*` resource). Per-service **operational** IRSA roles authored here too:
  report-service (S3 + Bedrock), notification-service (SQS + SES), ai_lambda (Textract + Bedrock + S3 read).
- `lablumen-terraform/outputs.tf` — `ecr_repository_urls`, `runtime_secret_arns`, `eso_irsa_role_arn` (+ SSM param names TBD).

**Lock / Verification Criteria (binary):**
- [x] Remote-state bootstrap authored (`bootstrap/` S3 + DynamoDB, both `prevent_destroy`).
- [x] Root `backend "s3"` block enabled, names match `bootstrap/variables.tf`.
- [x] ECR repos authored, **singular** names matching `lablumen-k8s` values + `lablumen-app` paths.
- [x] Secrets Manager **containers** authored with **no** `aws_secretsmanager_secret_version` (no values in state).
- [x] `ssm.tf` authored: `/lablumen/config/{reports-bucket,sqs-url,cognito-user-pool-id,cognito-app-client-id,ses-sender,bedrock-embed-model,bedrock-text-model,region}` published from module outputs.
- [ ] `bootstrap` applied: `terraform -chdir=bootstrap apply` → bucket + table exist.
- [ ] Root migrated to remote backend: `terraform init -migrate-state` succeeds; state in S3, lock in DynamoDB.
- [x] `terraform fmt -check -recursive` clean **and** `terraform validate` clean.
- [ ] `terraform plan` clean (ECR + Secrets Manager + SSM only; no drift).
- [x] `ssm.tf` authored: `/lablumen/config/{reports-bucket,sqs-url,cognito-user-pool-id,cognito-app-client-id,ses-sender,bedrock-embed-model,bedrock-text-model,region,presigned-url-ttl,cors-origins}` published from module outputs (already listed above but repeated for clarity).
- [x] ESO IRSA role authored (`aws_iam_role.eso`); trust principal is `system:serviceaccount:external-secrets:external-secrets`; SM + SSM policy scoped to named paths only (no `*`).
- [x] Per-service operational IRSA roles authored: report-service (S3 + Bedrock), notification-service (SQS + SES), ai_lambda (Textract + Bedrock + S3). These roles grant **operational** AWS actions only — they do NOT grant SM/SSM reads.
- [x] `eso_irsa_role_arn` output in `outputs.tf`.
- [ ] Secrets/params readable only by intended principals (no broad `*` resource on reads beyond Bedrock).
- [ ] Reviewed & immutable (state backend + secret/param scaffolding signed off).

---

## Phase 1 — Network  **[NOT STARTED]**

**Build Inputs:** P0 LOCKED; `modules/network` (two-tier today); blueprint §4.

**Resource Vectors:**
- `modules/network` — add **`database_subnets`** list (`10.0.201.0/24`, `10.0.202.0/24`), no NAT/IGW route.
- `variables.tf` / `terraform.tfvars` — `database_subnets` variable + values.
- VPC **endpoints** (private app subnets, SG `443` from VPC CIDR):
  - Gateway: `s3`.
  - Interface: `secretsmanager`, **`ssm`**, `bedrock-runtime`, `textract`, `ecr.api`, `ecr.dkr`, `logs`, `sqs`.
  - Note: `secretsmanager` + `ssm` endpoints are also required by the **ESO controller** (P4) to fetch config
    without NAT. These must be present and have private DNS enabled before ESO is deployed.
- Keep `single_nat_gateway = true`. Preserve subnet tags (`kubernetes.io/role/*-elb`, `karpenter.sh/discovery`).

**Lock / Verification Criteria (binary):**
- [x] Three subnet tiers exist per AZ (public / private app / isolated DB); CIDRs match blueprint §4.2.
- [x] Isolated DB subnets have **no** default route (no NAT, no IGW).
- [x] All 9 endpoints present; **`ssm` + `secretsmanager`** interface endpoints resolve in-VPC (private DNS on).
- [x] Endpoint SG allows `443` from VPC CIDR only.
- [x] Subnet tags verified (`elb`/`internal-elb`/`karpenter.sh/discovery`); DB subnets untagged for k8s/karpenter.
- [ ] `terraform plan/apply` clean; route tables correct per tier.

---

## Phase 2 — Data Platform  **[NOT STARTED]**

**Build Inputs:** P1 LOCKED; `modules/data` (RDS in private app subnets today); blueprint §4.1, §7.2.

**Resource Vectors:**
- `modules/data` — move RDS into the **isolated DB subnet group** (`database_subnets`).
- RDS SG — ingress `5432` **only** from EKS node SG and (later) `lablumen-ai-lambda-sg`.
- RDS master credentials remain **RDS-managed** (Secrets Manager, RDS-owned).
- Human step: compose `DATABASE_URL` from endpoint + master secret → write into `lablumen/app/database-url`.

**Lock / Verification Criteria (binary):**
- [ ] RDS resides in isolated DB subnet group; **no public route**, `publicly_accessible = false`.
- [ ] `5432` reachable from app tier SG only; denied from elsewhere.
- [ ] `rds_endpoint` + `rds_master_user_secret_arn` outputs present.
- [ ] `DATABASE_URL` value populated in Secrets Manager (out-of-band) and resolvable by an IRSA test principal.
- [ ] Tier isolation confirmed (connectivity test from app subnet succeeds, from public fails).

---

## Phase 3 — EKS Control Plane  **[NOT STARTED]**

**Build Inputs:** P2 LOCKED; `modules/eks` (MNG desired2/max4 + Karpenter submodule today); blueprint §5.

**Resource Vectors:**
- `modules/eks` — trim bootstrap MNG to **`min1 / desired1 / max2`** `t3.large` (cluster-critical pods only).
- Karpenter controller IAM role, node IAM role + instance profile, interruption SQS queue.
- Discovery tags on subnets + node SG (`karpenter.sh/discovery = lablumen-eks`).
- `modules/identity` — per-service **operational** IRSA roles wired to EKS OIDC provider (report-service: S3 +
  Bedrock; notification-service: SQS + SES; ai_lambda: Textract + Bedrock + S3; appointment-service: identity only).
  ESO IRSA role wired to OIDC provider (authored in P0, activated here). LBC + Karpenter controller IRSA.

**Lock / Verification Criteria (binary):**
- [ ] EKS `1.31` reachable; bootstrap nodes `Ready` (1 node, t3.large).
- [ ] CoreDNS / control pods schedulable on bootstrap MNG.
- [ ] Karpenter controller IAM + instance profile + interruption queue exist; discovery tags present.
- [ ] Per-service operational IRSA roles wired to OIDC provider; each grants only its operational actions (S3 / Bedrock / SQS / SES) — no SM/SSM config reads (ESO owns that).
- [ ] ESO IRSA role (`lablumen-eso`) wired to OIDC provider; trust principal matches `external-secrets/external-secrets` SA.
- [ ] OIDC provider wired (`module.eks.oidc_provider_arn` → `module.identity`).
- [ ] ESO IRSA smoke-test: assume ESO role via web-identity → confirm `secretsmanager:GetSecretValue lablumen/app/database-url` and `ssm:GetParameter /lablumen/config/sqs-url` succeed via VPC endpoints (no NAT). Verify operational IRSA roles deny SM/SSM reads.

---

## Phase 4 — Gateway CRDs + Add-ons + ESO  **[NOT STARTED]**

**Build:** Bootstrap Gateway API CRDs via Terraform Helm provider **first**; install ArgoCD; sync `platform-addons`
(LBC, Karpenter, **k-gateway**, **ESO**; **no CSI**). Once ESO pods are `Running`, deploy `ClusterSecretStore`
pointing to ESO IRSA + us-east-1, then deploy per-service `ExternalSecret` manifests. Verify K8s Secrets are
materialised before advancing to P5.

**Vectors:**
- `lablumen-k8s/platform-addons/*`, `root-app.yaml`
- `lablumen-k8s/platform-addons/external-secrets.yaml` — ArgoCD `Application` for ESO Helm chart
  (`external-secrets/external-secrets`; SA annotation: `eso_irsa_role_arn` from Terraform output)
- `lablumen-k8s/config/cluster-secret-store.yaml` — `ClusterSecretStore "lablumen-aws"` (aws provider; ESO IRSA)
- `lablumen-k8s/config/external-secrets/appointment-service.yaml` — `ExternalSecret` (SM: database-url; SSM: region, cognito-*)
- `lablumen-k8s/config/external-secrets/report-service.yaml` — `ExternalSecret` (SM: database-url; SSM: region, cognito-*, reports-bucket, bedrock-*)
- `lablumen-k8s/config/external-secrets/notification-service.yaml` — `ExternalSecret` (SM: database-url; SSM: region, sqs-url, ses-sender)

**Lock:**
- [ ] `kubectl get crd` shows `gateway.networking.k8s.io` and `externalsecrets.external-secrets.io`
- [ ] ArgoCD healthy; K-Gateway control plane up; CRDs exist before any `HTTPRoute`
- [ ] ESO pods `Running` in `external-secrets` namespace
- [ ] `ClusterSecretStore "lablumen-aws"` status: `Ready`
- [ ] Each service `ExternalSecret` status: `SecretSynced`; `kubectl get secret appointment-service-config -n lablumen -o json` shows correct keys
- [ ] Test pod with `envFrom.secretRef: appointment-service-config` resolves `DATABASE_URL` to the actual DSN (non-empty, not a path placeholder)
- [ ] `secrets-store-csi-driver` removed from the add-on tree

## Phase 5 — Karpenter Live  **[NOT STARTED]**

**Build:** `NodePool` + `EC2NodeClass` via GitOps; scale-test; retire static nodes. **Vectors:**
`lablumen-k8s/platform-addons/karpenter/*`. **Lock:** [ ] pending pods trigger `CreateFleet`; [ ] consolidation
works; [ ] spot interruption drains cleanly; [ ] bootstrap MNG at min.

## Phase 6 — AI Bridge  **[NOT STARTED]**

**Build:** Wire `ai_lambda` `VpcConfig` + `lablumen-ai-lambda-sg` + RDS ingress rule; resolve `lambda_source_path`
(package zip from `lablumen-app/serverless/ai-service/` in CI, consumed as artifact).
**Vectors:** `modules/storage`, `modules/data` SG, `lablumen-app/serverless/ai-service/`, `lablumen-terraform/main.tf`
(uncomment VpcConfig).
**Pre-condition already satisfied:** Inline AI pipeline (ingestion.py, textract.py, chunking.py, summarize()) was
stripped from `report-service` on 2026-06-20 — this sub-task is **DONE**.
**Lock:**
- [ ] Upload → S3 event → Lambda invocation confirmed (CloudWatch log)
- [ ] Lambda → Textract → Bedrock → pgvector write confirmed (row in `report_embeddings`)
- [ ] `report-service` `/chat` endpoint reads vectors populated by Lambda (not inline)
- [ ] Lambda resolves `DATABASE_URL` from Secrets Manager (or ESO-managed Secret via init container pattern if in-cluster) at runtime
- [ ] `lambda_source_path` resolved: CI packages `lablumen-app/serverless/ai-service/` zip; Terraform consumes as artifact
- [ ] `modules/storage` bucket encryption corrected from KMS to SSE-S3 (open drift from P0)

## Phase 7 — Repo Consolidation + Delivery  **[NOT STARTED]**

**Build:** Consolidate to 3 repos; CI **path filters** in `lablumen-app` (`paths: ['backend/<svc>/**']`) build+push
per-service images to ECR and PR-bump `image.tag` in `lablumen-k8s`; repoint `root-app` `repoURL` → `lablumen-k8s`.
**Lock:** [ ] each path-filtered workflow builds/tests its workload; [ ] images land in ECR with git-SHA tags;
[ ] ArgoCD tracks `lablumen-k8s`; [ ] monorepo frozen.

## Phase 8 — Routing Cutover  **[NOT STARTED]**

**Build:** Deploy frontend (`lablumen-app/frontend`); shared-ALB → K-Gateway; per-service `HTTPRoute`s (replace
per-service Ingress); verify charts carry no hardcoded `Secret`/`ConfigMap` data — only `envFrom.secretRef` pointing
to ESO-managed Secrets is permitted.
**Lock:**
- [ ] `/api/v1/reports` + `/api/v1/` resolve via single ALB → K-Gateway → correct service
- [ ] One public ALB only; HTTPRoutes authoritative
- [ ] No chart in `lablumen-k8s` contains a `Secret` manifest with a `data:` block or a `ConfigMap` with env values
- [ ] All `envFrom.secretRef` targets resolve to an ESO-managed Secret (confirmed via `ExternalSecret` status)
- [ ] `envSecretName` values and legacy `envFrom.secretRef` pointing to manually-managed Secrets removed

## Phase 9 — Decommission  **[NOT STARTED]**

**Build:** Cut DNS to new ALB; drain compose; archive EC2 stack. **Lock:** [ ] full E2E on EKS; [ ] no traffic to
compose; [ ] `docker-compose.yml` retired; [ ] EKS is sole runtime.

---

## Cross-phase open drift (carry forward until resolved)

- **`lambda_source_path`** (`lablumen-terraform/main.tf`) points at `../serverless/...` — invalid across the
  `lablumen-terraform` / `lablumen-app` repo split. Resolve in **P6** (CI-built Lambda zip artifact).
- **`modules/storage` bucket** uses `aws:kms`; blueprint mandates **SSE-S3**. Switch in **P6** (storage work).
- **`lablumen-k8s` charts** still carry CSI-era `envSecretName` and the `secrets-store-csi-driver` addon.
  Remove CSI driver during **P4** (add-ons). Migrate any remaining `envFrom.secretRef` pointing to manually-managed
  Secrets to ESO-managed Secrets during **P8** (chart cleanup). Ref: `SECRETS_AND_CONFIG.md §7`.
- **Inline AI removal from `report-service`** — **DONE 2026-06-20** (ingestion.py, textract.py, chunking.py,
  summarize() removed; Lambda at `lablumen-app/serverless/ai-service/` is the sole ingestion path).
