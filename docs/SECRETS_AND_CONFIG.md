# Secrets & Configuration Architecture — ESO Runtime Injection

> **Document type:** Locked architecture reference for runtime configuration.
> **Model:** Application workloads are **environment-blind**. Configuration and credentials are injected at runtime
> by the **External Secrets Operator (ESO)**, which bridges AWS (Secrets Manager + SSM Parameter Store) to
> ephemeral Kubernetes `Secret` objects inside the cluster. Apps read plain environment variables — no AWS SDK
> config calls, no hardcoded values in git, no CSI mounts.
> **Repos:** `lablumen-terraform` (provisions AWS sinks + ESO IRSA role), `lablumen-k8s` (ESO manifests, config-blind),
> `lablumen-app` (consumes env vars via pydantic-settings — unchanged from compose). Docs in `extras`.
> See `EKS_MIGRATION_BLUEPRINT.md` §2 / §6.4 and `MIGRATION_STATE_LEDGER.md` P0 / P3 / P4.

---

## 0. The blank-image principle

Every microservice reads its settings from **environment variables** via `pydantic-settings`
(`lablumen-app/backend/*/app/config.py`); baked-in defaults exist **only for local `.env` development**. In-cluster,
nothing sensitive is compiled into the image and no config is hardcoded by Kubernetes.

The same immutable, git-SHA-tagged image runs unchanged in any environment. What changes between environments is
**only what ESO injects as env vars at pod boot** — never the image, never a chart value (beyond the image tag and
the `secretRef` name). The injection mechanism is transparent to the application: from the container's perspective
it is reading the same `DATABASE_URL` env var it always has.

Consequence: a leaked image discloses nothing. A leaked `lablumen-k8s` git history discloses nothing — the
`ExternalSecret` manifests carry only AWS path references (e.g. `lablumen/app/database-url`), never values.
Credentials and topology live exclusively in AWS Secrets Manager + SSM Parameter Store, fetched by ESO via IRSA.

---

## 1. The two-lane model

| Lane | AWS sink | ESO sync mechanism | Refresh interval | Examples |
|---|---|---|---|---|
| 🔒 **Sensitive** | AWS Secrets Manager | `secretsmanager` provider → K8s `Secret` | `1h` | `DATABASE_URL`, SMTP creds, internal service keys |
| 📍 **Non-sensitive** | SSM Parameter Store | `parameterStore` provider → K8s `Secret` | `12h` | SQS URL, S3 bucket, Cognito IDs, Bedrock model IDs, region |

Both lanes are resolved **by ESO**, not by the application. `lablumen-k8s` carries neither lane's values — it carries
only the AWS path references inside `ExternalSecret` manifests.

### 1.1 Naming conventions (Terraform-owned)

| Lane | Path convention | Populated by |
|---|---|---|
| Secrets Manager | `lablumen/app/<name>` (e.g. `lablumen/app/database-url`) | Terraform creates **empty container** (`aws_secretsmanager_secret`, no `_version`); human writes value out-of-band |
| SSM Parameter Store | `/lablumen/config/<key>` (e.g. `/lablumen/config/sqs-url`) | Terraform publishes value directly from module outputs (`aws_ssm_parameter`) |

---

## 2. ESO topology

### 2.1 End-to-end data flow

```
lablumen-terraform
  ├── aws_secretsmanager_secret  "lablumen/app/database-url"  ← EMPTY container (value out-of-band, never in tfstate)
  ├── aws_ssm_parameter          "/lablumen/config/sqs-url"   ← populated from module.messaging output
  ├── aws_ssm_parameter          "/lablumen/config/..."       ← all non-sensitive infra constants
  └── aws_iam_role.eso           ← ESO IRSA role; trust: ESO ServiceAccount in lablumen-k8s

lablumen-k8s
  ├── ClusterSecretStore "lablumen-aws"   ← cluster-scoped; aws provider; roleArn → eso IRSA; region us-east-1
  └── ExternalSecret "appointment-service-config"
      ExternalSecret "report-service-config"
      ExternalSecret "notification-service-config"
        ↑ these carry SM/SSM paths only — no values — repo stays blind

ESO controller (external-secrets namespace, in-cluster)
  └── authenticates with AWS via IRSA (web-identity token bound to ESO ServiceAccount)
      reads SM + SSM via secretsmanager + ssm VPC interface endpoints (no NAT)
      materialises ephemeral Kubernetes Secrets (cluster memory only; not in etcd-backed PVs)

App pod
  └── envFrom:
        secretRef:
          name: appointment-service-config   ← ESO-managed Secret
      ↓
      DATABASE_URL, AWS_REGION, COGNITO_USER_POOL_ID, …   ← plain env vars
      ↓
      pydantic-settings (Settings class in config.py)   ← zero code change from compose model
```

### 2.2 `ClusterSecretStore` spec

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: lablumen-aws
spec:
  provider:
    aws:
      service: SecretsManager        # default; overridden per-entry for SSM
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

> ESO's Helm chart creates the `external-secrets` ServiceAccount. The chart's `serviceAccount.annotations` must
> include `eks.amazonaws.com/role-arn: <eso_irsa_role_arn>` (sourced from `lablumen-terraform` output).

### 2.3 `ExternalSecret` spec pattern

One `ExternalSecret` per service, co-located with its Deployment in `lablumen-k8s`. Example for
`appointment-service` (the pattern repeats for the other services):

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: appointment-service-config
  namespace: lablumen
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: lablumen-aws
    kind: ClusterSecretStore
  target:
    name: appointment-service-config   # name of the K8s Secret ESO will create/own
    creationPolicy: Owner
    deletionPolicy: Retain
  data:
    # ── Sensitive lane (Secrets Manager) ──────────────────────────
    - secretKey: DATABASE_URL                    # env var name in the pod
      remoteRef:
        key: lablumen/app/database-url           # SM secret name
    # ── Non-sensitive lane (SSM Parameter Store) ──────────────────
    - secretKey: AWS_REGION
      remoteRef:
        key: /lablumen/config/region
        decodingStrategy: None
      sourceRef:
        storeRef:
          name: lablumen-aws
          kind: ClusterSecretStore
        # override provider for SSM
        generatorRef: {}             # (use dataFrom with parameterStore provider — see note below)
```

> **SSM via ESO:** For Parameter Store entries, use the `ParameterStore` generator or a second `ClusterSecretStore`
> with `service: ParameterStore`. The cleanest production approach: one `ClusterSecretStore` per provider type
> (`lablumen-aws-sm` for Secrets Manager, `lablumen-aws-ssm` for Parameter Store), then each `ExternalSecret`
> references the appropriate store per entry. Both stores use the same ESO IRSA role.

### 2.4 Config key → env var mapping

| Setting | Service(s) | Lane | AWS Path | Env var (pod) |
|---|---|---|---|---|
| `database_url` | appointment, report | 🔒 SM | `lablumen/app/database-url` | `DATABASE_URL` |
| `aws_region` | all | 📍 SSM | `/lablumen/config/region` | `AWS_REGION` |
| `cognito_user_pool_id` | appointment, report | 📍 SSM | `/lablumen/config/cognito-user-pool-id` | `COGNITO_USER_POOL_ID` |
| `cognito_app_client_id` | appointment, report | 📍 SSM | `/lablumen/config/cognito-app-client-id` | `COGNITO_APP_CLIENT_ID` |
| `reports_s3_bucket` | report | 📍 SSM | `/lablumen/config/reports-bucket` | `REPORTS_S3_BUCKET` |
| `notifications_queue_url` | notification | 📍 SSM | `/lablumen/config/sqs-url` | `NOTIFICATIONS_QUEUE_URL` |
| `ses_sender_email` | notification | 📍 SSM | `/lablumen/config/ses-sender` | `SES_SENDER_EMAIL` |
| `bedrock_embed_model_id` | report, ai_lambda | 📍 SSM | `/lablumen/config/bedrock-embed-model` | `BEDROCK_EMBED_MODEL_ID` |
| `bedrock_text_model_id` | report, ai_lambda | 📍 SSM | `/lablumen/config/bedrock-text-model` | `BEDROCK_TEXT_MODEL_ID` |
| `presigned_url_ttl_seconds` | report | 📍 SSM | `/lablumen/config/presigned-url-ttl` | `PRESIGNED_URL_TTL_SECONDS` |
| `cors_origins` | all | 📍 SSM | `/lablumen/config/cors-origins` | `CORS_ORIGINS` |

> **Bedrock policy lock:** `bedrock_text_model_id = "amazon.nova-lite-v1:0"` is canonical. Nova 2 Lite requires a
> cross-region inference profile blocked by org SCP `p-rn6vr8ok` (us-east-1 only). This value is published to SSM
> by Terraform and consumed by ESO — no hardcoding anywhere.

---

## 3. Authentication boundary

### 3.1 ESO IRSA (the only config-reading identity)

```
ESO controller pod
  → ServiceAccount: external-secrets (namespace: external-secrets)
  → Annotation: eks.amazonaws.com/role-arn = arn:aws:iam::130290476321:role/lablumen-eso
  → AWS STS (web-identity) → temp creds
  → secretsmanager:GetSecretValue on arn:aws:secretsmanager:us-east-1:130290476321:secret:lablumen/app/*
  → ssm:GetParameter + ssm:GetParametersByPath on /lablumen/config/*
  → via secretsmanager + ssm VPC interface endpoints (no NAT)
```

ESO's IRSA role is provisioned by `lablumen-terraform` (e.g. `iam.tf`) with the OIDC trust policy bound to the
`external-secrets/external-secrets` ServiceAccount. Its policy grants **only** the two config-read actions on the
scoped paths — no `*` resource, no write permissions.

### 3.2 Per-service IRSA (operational AWS calls only — NOT for config)

App pods still carry their own IRSA for **operational** AWS service calls. These are entirely separate from config
injection:

| Service | Operational IRSA grants |
|---|---|
| `report-service` | `s3:GetObject` / `s3:PutObject` on `lablumen-reports-*`; `bedrock:InvokeModel` (Titan embed + Nova Lite) |
| `notification-service` | `sqs:ReceiveMessage` / `sqs:DeleteMessage` on `lablumen-notifications`; `ses:SendEmail` |
| `ai_lambda` | `textract:DetectDocumentText`; `bedrock:InvokeModel`; `s3:GetObject` on `lablumen-reports-*` |
| `appointment-service` | No operational AWS calls; ServiceAccount exists for K8s identity only |

Per-service operational IRSA roles are provisioned by `lablumen-terraform` (e.g. `modules/identity/`), wired to the
EKS OIDC provider, and annotated onto each service's `ServiceAccount` in `lablumen-k8s`.

### 3.3 Network path

All ESO → AWS calls go through the **`secretsmanager` and `ssm` interface VPC endpoints** provisioned in P1 —
private DNS resolves in-VPC, traffic never leaves the AWS network and does not consume NAT gateway bandwidth.

---

## 4. Sync policy and failure behaviour

| Policy | Setting | Reason |
|---|---|---|
| `creationPolicy: Owner` | ESO creates and owns the K8s Secret | Clean lifecycle: ESO reconciles on drift |
| `deletionPolicy: Retain` | K8s Secret persists if `ExternalSecret` is deleted | Prevents boot outage during GitOps mistakes |
| `refreshInterval` (sensitive) | `1h` | Picks up rotated SM secrets within one hour without pod restart |
| `refreshInterval` (non-sensitive) | `12h` | SSM params are infrastructure constants; changes require a redeploy anyway |

**Resilience:** If ESO cannot reach AWS at sync time, the previously materialised K8s Secret remains intact. Pods
that are already running continue unaffected. New pods starting during an outage will still boot successfully using
the last-synced Secret. This is a significant improvement over the previous Direct SDK model where a transient AWS
API failure at pod startup caused a `sys.exit(1)` and a crash loop.

---

## 5. Repository boundary enforcement

| Repo | Responsibility | MUST NOT contain |
|---|---|---|
| `lablumen-terraform` | `aws_secretsmanager_secret` containers (**no** `_version`); `aws_ssm_parameter` (populated from outputs); ESO IRSA role; per-service operational IRSA roles | Secret **values** in state; `aws_secretsmanager_secret_version` |
| `lablumen-k8s` | `ClusterSecretStore`; `ExternalSecret` manifests (paths only); `ServiceAccount` with operational IRSA annotations; Deployments with `envFrom.secretRef` | Raw values, passwords, API keys, `ConfigMap` env blocks, `Secret` manifests with `data:` |
| `lablumen-app` | Application source; `config.py` reads env vars via pydantic-settings (unchanged); operational `boto3` calls (S3 / Bedrock / SQS / SES) | Hardcoded creds; AWS SDK config bootstrap code; baked non-default config |
| `extras` | Architecture docs, migration ledger | Code, credentials, tfstate |

`lablumen-k8s` is **permanently configuration-blind in terms of values**: `ExternalSecret` manifests name the
*paths* in AWS, never the credentials themselves. The only non-default value a Deployment manifest carries is
the container `image.tag` (a delivery concern, not a config concern).

---

## 6. Pod injection — Deployment spec pattern

```yaml
# lablumen-k8s/charts/appointment-service/templates/deployment.yaml
containers:
  - name: appointment-service
    image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
    envFrom:
      - secretRef:
          name: appointment-service-config   # ESO-managed Secret
    # No env: blocks with hardcoded values.
    # No configMapRef blocks.
    # DATABASE_URL, AWS_REGION, COGNITO_*, etc. arrive via the secretRef above.
```

The `envFrom.secretRef` name must match the `ExternalSecret.spec.target.name` in `lablumen-k8s/config/external-secrets/`.

---

## 7. What changed vs. the previous Direct SDK design

| Area | Previous (Direct SDK) | Current (ESO) |
|---|---|---|
| Config-reading identity | Per-service IRSA; each app called `boto3` at startup | ESO controller holds one cluster-level IRSA; app pods call nothing |
| App code change required? | Yes — `config.py` needed TTL-cached `get_secret_value` + `get_parameter` + retry/backoff + `sys.exit(1)` | **No** — `config.py` is unchanged; pydantic-settings reads env vars exactly as in docker-compose |
| Secret visibility in git | `lablumen-k8s` config-blind (correct then too) | Same — `ExternalSecret` paths in git, values never in git |
| Failure mode at boot | `sys.exit(1)` crash loop if AWS unreachable | Last-synced K8s Secret persists; pod starts on stale (but valid) config |
| K8s add-on required | None (app pulled directly) | ESO Helm chart (`external-secrets/external-secrets`); installed via ArgoCD in P4 |
| `secrets-store-csi-driver` | Was being removed in P4 | Still removed in P4; ESO replaces it |
| Per-service IRSA | SM + SSM reads + operational calls | Operational AWS calls only (S3 / Bedrock / SQS / SES); SM+SSM reads are ESO's job |
| `lablumen-k8s` new manifests | None for config | `ClusterSecretStore`, `ExternalSecret` per service |
