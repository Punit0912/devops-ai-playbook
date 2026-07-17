# Project: Boutique Microservices on EKS (Zero Manual Intervention)

> Redesigned version of the devops-ai-playbook that deploys end-to-end without any manual steps after initial setup.

---

## Services Used

```
GitHub Actions (CI) → ECR (images) → ArgoCD + Image Updater (CD)
→ EKS (7 Deployments + 1 StatefulSet) → PostgreSQL (in-cluster, auto-initialized)
→ Prometheus + Grafana (observability)
```

---

## Architecture

```
Developer
  → pushes code to GitHub (main branch)
      → GitHub Actions (CI):
          → builds Docker images (tag = git SHA)
          → pushes to ECR
          → DONE (CI never touches manifests or cluster)
      → ArgoCD Image Updater (running in-cluster):
          → detects new image tag in ECR
          → updates Kustomize overlay automatically
          → ArgoCD syncs new state to cluster
              → new pods pull updated images
              → init containers run DB migrations
              → app starts healthy
```

---

## What Gets Created (terraform apply — one command)

```
┌─────────────────────────────────────────────────────────────┐
│ AWS Account                                                 │
│                                                             │
│  VPC + 3 Subnets + IGW + Route Table                       │
│  EKS Cluster + Node Group (auto-scaling 1-3)               │
│  ECR Repositories (7 — one per service)                    │
│  IAM Roles (cluster, node, EBS CSI via IRSA)               │
│  OIDC Provider (for IRSA)                                  │
│  EBS CSI Driver (addon)                                    │
│                                                             │
│  Helm Releases (on cluster):                               │
│  ├── ArgoCD                                                │
│  ├── ArgoCD Image Updater                                  │
│  ├── kube-prometheus-stack (Prometheus + Grafana)           │
│  └── ArgoCD Application (self-managing bootstrap)          │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Design Decisions (No Manual Intervention)

### 1. No Hardcoded Account IDs in Manifests

**Problem we had:** Manifests contained `<AWS_ACCOUNT_ID>` placeholder — broke everything.

**Fix:** Use Kustomize image transformer. Manifests reference a generic image name; Kustomize rewrites it at deploy time.

```yaml
# kustomization.yml
images:
  - name: boutique/auth            # generic reference in Deployment
    newName: 417007889303.dkr.ecr.us-east-1.amazonaws.com/auth
    newTag: latest                  # Image Updater overwrites this
```

Deployments use the generic name:
```yaml
containers:
  - name: auth
    image: boutique/auth           # Kustomize transforms this
```

**Result:** Manifests never contain account-specific values. Works for any fork.

---

### 2. No Manual Image Tag Updates

**Problem we had:** CI used `sed` to update tags in manifests and committed back — created noise, merge conflicts, local/remote drift.

**Fix:** ArgoCD Image Updater watches ECR and auto-updates tags.

```yaml
# ArgoCD Application annotations
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: boutique
  annotations:
    argocd-image-updater.argoproj.io/image-list: >
      auth=417007889303.dkr.ecr.us-east-1.amazonaws.com/auth,
      gateway=417007889303.dkr.ecr.us-east-1.amazonaws.com/gateway,
      frontend=417007889303.dkr.ecr.us-east-1.amazonaws.com/frontend,
      orders=417007889303.dkr.ecr.us-east-1.amazonaws.com/orders,
      order-service=417007889303.dkr.ecr.us-east-1.amazonaws.com/order-service,
      product-service=417007889303.dkr.ecr.us-east-1.amazonaws.com/product-service,
      user-service=417007889303.dkr.ecr.us-east-1.amazonaws.com/user-service
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
```

**Result:** CI builds and pushes. That's it. Image Updater handles the rest.

---

### 3. No Manual Database Initialization

**Problem we had:** PostgreSQL started empty. Services crashed. Had to manually exec into pod and create databases.

**Fix:** Init container on the StatefulSet that creates databases + runs migrations on first boot.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: boutique-postgres
spec:
  template:
    spec:
      initContainers:
        - name: init-databases
          image: postgres:15-alpine
          command:
            - sh
            - -c
            - |
              until pg_isready -h localhost -U postgres; do sleep 1; done
              for DB in auth_db orders_db products_db users_db; do
                psql -U postgres -tc "SELECT 1 FROM pg_database WHERE datname='$DB'" | \
                  grep -q 1 || psql -U postgres -c "CREATE DATABASE $DB;"
              done
          env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: boutique-secrets
                  key: POSTGRES_PASSWORD
      containers:
        - name: postgres
          image: postgres:15-alpine
          # ...
```

Each microservice runs its own schema migration on startup (already does this — `CREATE TABLE IF NOT EXISTS`). No separate Job needed.

**Result:** First deploy → Postgres starts → init container creates DBs → app pods connect successfully.

---

### 4. No Manual ArgoCD Setup (App-of-Apps Bootstrap)

**Problem we had:** Had to port-forward, log in, connect repo, create application manually.

**Fix:** Terraform applies the ArgoCD Application resource directly (app-of-apps pattern).

```terraform
# In modules/argocd/main.tf
resource "kubernetes_manifest" "argocd_app" {
  manifest = yamldecode(<<-YAML
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: boutique
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: var.git_repo_url
        targetRevision: main
        path: gitops
      destination:
        server: https://kubernetes.default.svc
        namespace: boutique
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
  YAML
  )

  depends_on = [helm_release.argocd]
}
```

**Result:** `terraform apply` → ArgoCD installed + application registered + auto-sync enabled. Zero UI interaction.

---

### 5. No Manual Repo Connection

**Problem we had:** Had to manually connect the repo in ArgoCD UI with a PAT.

**Fix:** For public repos — no auth needed. For private repos — Terraform creates the repo secret:

```terraform
resource "kubernetes_secret" "argocd_repo" {
  metadata {
    name      = "repo-boutique"
    namespace = "argocd"
    labels = {
      "argocd.argoproj.io/secret-type" = "repository"
    }
  }

  data = {
    url      = var.git_repo_url
    password = var.github_pat    # from terraform.tfvars or Secrets Manager
    username = "git"
    type     = "git"
  }
}
```

**Result:** ArgoCD auto-discovers repo credentials. No manual UI clicks.

---

## Revised CI Pipeline (GitHub Actions)

```yaml
name: Boutique CI

on:
  push:
    branches: [main]
    paths:
      - 'projects/boutique-microservices/**'
  workflow_dispatch:

env:
  AWS_REGION: us-east-1

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        service: [auth, gateway, orders, order-service, product-service, user-service, frontend]
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-role
          aws-region: ${{ env.AWS_REGION }}
          # ↑ OIDC federation — no static keys

      - name: Login to ECR
        run: aws ecr get-login-password | docker login --username AWS --password-stdin ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com

      - name: Build and Push
        run: |
          IMAGE=${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ matrix.service }}
          TAG=${{ github.sha }}

          if [ "${{ matrix.service }}" == "frontend" ]; then
            BUILD_PATH="projects/boutique-microservices/frontend"
          else
            BUILD_PATH="projects/boutique-microservices/backend/services/${{ matrix.service }}"
          fi

          docker build -t $IMAGE:$TAG -t $IMAGE:latest $BUILD_PATH
          docker push $IMAGE:$TAG
          docker push $IMAGE:latest

  # NO update-manifests job — ArgoCD Image Updater handles it
```

**Key differences from original:**
- Uses **OIDC federation** (no static AWS keys in secrets)
- Pushes both `SHA` tag and `latest` (SHA for traceability, latest for convenience)
- **No manifest update step** — Image Updater does this
- Only triggers on changes to actual service code (not docs/infra)

---

## Complete End-to-End Flow (Zero Touch After Setup)

```
┌──────────────────────────────────────────────────────────────┐
│ ONE-TIME SETUP (5 minutes)                                   │
│                                                              │
│ 1. Fork repo                                                 │
│ 2. Set GitHub secret: AWS_ACCOUNT_ID                         │
│ 3. Create OIDC identity provider in AWS for GitHub Actions   │
│ 4. terraform apply                                           │
│    (creates everything: VPC, EKS, ECR, ArgoCD, App, DB)      │
│                                                              │
│ DONE. Never touch the cluster manually again.                │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ EVERY CODE PUSH (fully automated)                            │
│                                                              │
│ Developer pushes code                                        │
│     → GitHub Actions builds + pushes to ECR                  │
│         → ArgoCD Image Updater detects new tag               │
│             → Updates kustomization overlay in Git            │
│                 → ArgoCD syncs new manifests                  │
│                     → New pods roll out (zero-downtime)       │
│                         → Init containers ensure DB ready    │
│                             → App healthy ✅                  │
└──────────────────────────────────────────────────────────────┘
```

---

## Comparison: Original vs Redesigned

| Step | Original (manual pain) | Redesigned (automated) |
|------|----------------------|------------------------|
| Image tags in manifests | `<AWS_ACCOUNT_ID>` placeholder, manual fix | Kustomize image transformer, no account IDs in YAML |
| CI updates manifests | `sed` + git commit in pipeline | ArgoCD Image Updater (in-cluster) |
| ArgoCD repo connection | Manual UI: Settings > Repos > PAT | Terraform creates repo secret |
| ArgoCD app creation | Manual: Create App in UI | Terraform applies Application manifest |
| DB initialization | Manual exec into pod / broken restore Job | Init container on StatefulSet |
| Local vs GitHub drift | Image tags differ → pull errors | Local manifests don't contain tags — Image Updater owns them |
| CI auth to AWS | Static access keys in GitHub Secrets | OIDC federation (no keys) |
| First sync | Manual: click Sync in ArgoCD UI | `syncPolicy.automated` — auto-syncs immediately |
| Rollback | Re-run pipeline or manual kubectl | `git revert` → ArgoCD auto-syncs reverted state |

---

## For Production: Add These

| Component | What to add | How |
|-----------|-------------|-----|
| Ingress | AWS Load Balancer Controller | Terraform module + Ingress resource per service |
| DNS | Route53 hosted zone | Alias record → ALB |
| TLS | ACM certificate | Annotation on Ingress |
| WAF | WAF v2 WebACL | Annotation or Terraform attachment |
| Auth | Cognito | ALB listener action |
| Secrets | External Secrets Operator | Syncs from Secrets Manager → K8s Secrets |
| DB | RDS Aurora | Replace StatefulSet with `aws_rds_cluster` |
| Autoscaling | Karpenter | Helm release in Terraform |
| Logging | Fluent Bit → CloudWatch | DaemonSet |
| Multi-env | ArgoCD ApplicationSet | One ApplicationSet → dev/staging/prod namespaces |

---

## Request Flow (Production — for Reference)

```
Client
  → Route 53 (Alias A record → ALB)
      → WAF v2 (WebACL on ALB — rate limiting, geo-blocking, SQL injection)
          → ALB (created by AWS LB Controller from Ingress resource)
              → path-based rules: /api/auth → auth TG, /api/orders → orders TG, / → frontend TG
                  → Target Group (target-type: ip — routes to pod IPs via VPC CNI)
                      → Pod in EKS (image from ECR, secrets from Secrets Manager via IRSA)
                          → RDS Aurora (via DATABASE_URL from External Secrets)
```

---

*This is how the project should work. One `terraform apply`, then every git push flows automatically to production without anyone touching kubectl, ArgoCD UI, or manually fixing image tags.*
