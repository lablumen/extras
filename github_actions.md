# GitHub Actions

LabLumen uses a centralized CI/CD model. All reusable pipeline logic lives in `lablumen-shared`. Each service repository calls these workflows as thin wrappers — updating a workflow in `lablumen-shared` propagates to all services automatically on their next run.

---

## Shared Workflows

`lablumen-shared` contains three reusable workflows, all triggered via `workflow_call` (they cannot be run directly).

---

### Pull Request Gate — `service-pr.yml`

Runs on every pull request. All jobs must pass before a PR can merge.

| Job | What It Does |
|---|---|
| `lint-and-test` | Python: `ruff check` + `pytest` / Node: `npm ci` + `npm run build` |
| `sast` | SonarCloud static analysis — code quality and security vulnerability detection |
| `sca` | Snyk software composition analysis — known CVEs in `requirements.txt` or `package-lock.json` |
| `container-scan` | Builds the Docker image locally, scans all layers with Trivy — fails on CRITICAL or HIGH findings |

`sast`, `sca`, and `container-scan` run in parallel after `lint-and-test` succeeds.

---

### Dev Deploy — `service-build-push.yml`

Runs when a PR is merged to `main`.

1. Build the Docker image tagged with the 7-character git SHA (e.g., `abc1234`).
2. Run a Trivy gate — the image is not pushed if CRITICAL or HIGH vulnerabilities are found.
3. Push the image to ECR using GitHub OIDC. No static AWS credentials.
4. Check out `lablumen-k8s` and write the new SHA tag into `services/<name>/values-dev.yaml`.
5. Commit and push using a `git pull --rebase` retry loop (up to 5 attempts) to handle concurrent service builds writing to the same GitOps repo at the same time.
6. ArgoCD detects the commit and rolls out the new image to `lablumen-dev`.

---

### Production Promotion — `service-release.yml`

Runs when a GitHub Release is published on a service repository.

1. Retrieve the ECR image manifest for the SHA tag that was deployed to dev.
2. Create a new ECR tag with the release semver (e.g., `v1.2.0`) pointing to the same image layers — **no rebuild**.
3. Write the semver tag into `services/<name>/values-prod.yaml` in `lablumen-k8s`.
4. ArgoCD detects the commit and rolls out to the `lablumen` (production) namespace.

---

## How Service Repos Call These Workflows

Each service has a single `.github/workflows/ci.yml` that delegates to the right workflow based on the GitHub event type:

```yaml
jobs:
  pr:
    if: github.event_name == 'pull_request'
    uses: lablumen/lablumen-shared/.github/workflows/service-pr.yml@main
    with:
      service-name: appointment-service
      service-path: .
      sonar-organization: ${{ vars.SONAR_ORGANIZATION }}
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  build:
    if: github.event_name == 'push'
    uses: lablumen/lablumen-shared/.github/workflows/service-build-push.yml@main
    with:
      service-name: appointment-service
      ecr-repository: lablumen/appointment-service
      role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/lablumen-app-ci-ecr
    secrets:
      K8S_REPO_PAT: ${{ secrets.K8S_REPO_PAT }}

  release:
    if: github.event_name == 'release'
    uses: lablumen/lablumen-shared/.github/workflows/service-release.yml@main
    with:
      service-name: appointment-service
      ecr-repository: lablumen/appointment-service
      role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/lablumen-app-ci-ecr
      release-tag: ${{ github.event.release.tag_name }}
    secrets:
      K8S_REPO_PAT: ${{ secrets.K8S_REPO_PAT }}
```

The `if:` conditions ensure only one job runs per event.

---

## Security Design

**No static AWS credentials.** All AWS access uses GitHub OIDC. Each job exchanges a GitHub-signed JWT for temporary STS credentials that expire after one hour.

**Build-once, promote.** The Docker image built and scanned in dev is promoted to production by retagging its ECR manifest — not by rebuilding. The exact artifact that passed security gates in dev is what runs in production.

**Least-privilege IAM roles, one per purpose:**

| Role | Assumed By | What It Can Do |
|---|---|---|
| `lablumen-app-ci-ecr` | Any `lablumen/*` service repo | ECR push for backend service images |
| `lablumen-frontend-build` | Frontend repo only | ECR push for the frontend image |
| `lablumen-ai-lambda-deploy` | AI service repo only | SAM deploy for the Lambda function |
| `lablumen-tf-plan` | Terraform repo, any branch | Read-only plan access |
| `lablumen-tf-apply` | Terraform repo, `production` env only | Admin access for `terraform apply` |

---

## Terraform Pipeline

The `lablumen-terraform` repository has its own dedicated pipeline separate from the shared service workflows:

| Stage | Trigger | Details |
|---|---|---|
| Scan | PR or push to `main` | Checkov IaC security scan — findings uploaded to GitHub Security tab |
| Plan | PR or push to `main` | `terraform fmt` → validate → plan; Infracost cost estimate posted as a PR comment |
| Apply | Push to `main` after human approval | `terraform apply` using the saved plan artifact; gated by the `production` GitHub Environment |
| Destroy | Manual trigger only | Requires a typed confirmation string; guarded teardown |

---

## AI Service Pipeline

The `lablumen-ai-service` uses its own pipeline (not the shared workflows) because it deploys via AWS SAM rather than ECR/ArgoCD:

| Trigger | What Happens |
|---|---|
| Pull request | `ruff` lint + `pytest` unit tests |
| Merge to `main` | `sam build --use-container` → `sam deploy` using the `lablumen-ai-lambda-deploy` OIDC role |

---

## Required GitHub Configuration

### Organization Variables

| Variable | Description |
|---|---|
| `AWS_ACCOUNT_ID` | 12-digit AWS account ID — used to construct IAM role ARNs in every workflow |
| `SONAR_ORGANIZATION` | SonarCloud organization key |

### Organization Secrets

| Secret | Description |
|---|---|
| `K8S_REPO_PAT` | Personal access token with `repo` write access to `lablumen-k8s` |
| `SONAR_TOKEN` | SonarCloud API token |
| `SNYK_TOKEN` | Snyk API token |

### Terraform Repository Secrets (repo-level)

| Secret | Description |
|---|---|
| `INFRACOST_API_KEY` | Infracost API key — cost estimates posted as PR comments |
| `BEDROCK_CROSS_ACCOUNT_ROLE_ARN` | Cross-account Bedrock IAM role ARN |

### Terraform Repository — GitHub Environment

Create an environment named **`production`** on `lablumen-terraform` with required reviewers. This is the manual gate that blocks `terraform apply` until a human approves the plan.
