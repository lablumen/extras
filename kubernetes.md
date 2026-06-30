# Kubernetes

EKS cluster `lablumen-eks` running Kubernetes 1.31 in `us-east-1`. All cluster state is managed via GitOps — the `lablumen-k8s` repository is the single source of truth for what runs in the cluster. No manual `kubectl apply` is used in day-to-day operations.

---

## Cluster

| Setting | Value |
|---|---|
| Cluster name | `lablumen-eks` |
| Region | `us-east-1` |
| Kubernetes version | 1.31 |
| Managed node group | `c7i-flex.large` (baseline, always-on) |
| Dynamic nodes | Karpenter (`t3.medium` / `t3.large`, on-demand) |
| Auth mode | EKS Access Entries |
| Provisioned by | Terraform (`modules/eks`) |

---

## Namespaces

| Namespace | Contents |
|---|---|
| `kube-system` | CoreDNS, kube-proxy, Metrics Server, AWS Load Balancer Controller, ExternalDNS, Karpenter |
| `argocd` | ArgoCD control plane |
| `external-secrets` | External Secrets Operator |
| `monitoring` | Prometheus, Grafana, AlertManager |
| `lablumen-dev` | All application services (SHA-based dev image tags) |
| `lablumen` | All application services (semver production image tags) |

---

## Running Workloads

The output below is from the live cluster (`kubectl get pods -A`):

![kubectl get pods -A](images/k8.png)

Application workloads running in both environments:

| Workload | Replicas (prod) | Notes |
|---|---|---|
| `appointment-service` | 2 | Owns all Alembic DB migrations |
| `report-service` | 2 | IRSA access to S3 and Bedrock |
| `notification-service` | 2 | IRSA access to SQS and SES |
| `frontend` (nginx) | 2 | Serves React SPA + reverse-proxies API calls |
| `redis` | 1 | In-cluster, no persistence — slot locking only |

---

## GitOps — ArgoCD

ArgoCD manages everything using the **App-of-Apps** pattern. One root `Application` (`bootstrap/root-app.yaml`) bootstraps all others in controlled sync waves:

| Wave | What Deploys |
|---|---|
| Wave 0 | Platform add-ons: ArgoCD self-manage, Karpenter, External Secrets Operator, AWS Load Balancer Controller, ExternalDNS, Metrics Server, Prometheus/Grafana |
| Wave 1 | Cluster-wide config: ESO ClusterSecretStores connecting to Secrets Manager and SSM |
| Wave 2 | All application services in both `lablumen-dev` and `lablumen` namespaces |

![ArgoCD Applications Dashboard](images/argocd.png)

All 22 ArgoCD applications show **Healthy / Synced** status. Changes to the cluster are made only by committing to `lablumen-k8s` — ArgoCD detects the change and reconciles automatically.

---

## Platform Add-ons

| Add-on | Purpose |
|---|---|
| Karpenter | Provisions EC2 nodes when pods are pending; consolidates underutilized nodes automatically |
| External Secrets Operator (ESO) | Syncs secrets from AWS Secrets Manager and SSM Parameter Store into Kubernetes Secrets |
| AWS Load Balancer Controller | Creates and manages the ALB from Kubernetes Ingress objects |
| ExternalDNS | Automatically creates Route 53 A records from Ingress hostnames |
| Prometheus + Grafana | Cluster and workload metrics — CPU, memory, pod counts, network throughput |
| Metrics Server | Provides resource metrics consumed by the Horizontal Pod Autoscaler |

---

## Microservice Helm Chart

All four stateless application services share a single reusable Helm chart (`charts/microservice`). Each service only supplies what is unique to it: image repository, ingress path, SSM parameter mappings, and replica count.

The chart generates the following objects for each service:

`Deployment` · `Service` · `Ingress` · `HorizontalPodAutoscaler` · `PodDisruptionBudget` · `ExternalSecret` · `ServiceAccount` · `NetworkPolicy`

---

## Secrets and Configuration

No secrets are committed to Git. External Secrets Operator pulls all configuration from AWS at runtime:

- **Sensitive values** — database URL from AWS Secrets Manager
- **Non-sensitive config** — Cognito pool ID, SQS URL, S3 bucket name, etc. from SSM Parameter Store under `/lablumen/config/*`

Each pod receives its config via `envFrom.secretRef`. No application code has an AWS SDK dependency for configuration fetching.

---

## Ingress and DNS

All external traffic enters via a single internet-facing ALB on HTTPS port 443, with TLS terminated using a wildcard ACM certificate (`*.rnld101.xyz`). Route 53 A records are managed automatically by ExternalDNS.

| Host | Service |
|---|---|
| `app.rnld101.xyz` | Frontend nginx — serves React SPA and proxies API calls to backend services |

---

## Monitoring

Prometheus and Grafana run in the `monitoring` namespace. Grafana admin credentials are synced from Secrets Manager via an ExternalSecret. The dashboard below shows the live cluster state:

![Grafana K8S Dashboard](images/grafana.png)

Key metrics tracked: node CPU and memory utilization, pod counts per namespace, workload health, and network throughput.

---

## Bootstrap

Once Terraform has created the EKS cluster, ArgoCD is bootstrapped with a single command:

```bash
aws eks update-kubeconfig --region us-east-1 --name lablumen-eks
cd lablumen-k8s
bash scripts/bootstrap-argocd.sh
```

This installs ArgoCD via Helm and applies `bootstrap/root-app.yaml`. ArgoCD then manages everything else automatically.
