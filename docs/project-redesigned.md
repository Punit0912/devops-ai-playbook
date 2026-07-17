# Project: Boutique Microservices on EKS (Zero Manual Intervention)

> Redesigned version of the devops-ai-playbook that deploys end-to-end without any manual steps after initial setup.

---

## Who Does What — In Order

### Phase 1: Developers (Application Team)

```
1. Write application code (React frontend, Node.js backend services)
2. Write Dockerfiles for each service (multi-stage builds)
3. Create docker-compose.yml for local development
4. Test locally: docker-compose up → verify services communicate
5. Write unit/integration tests
6. Push code to GitHub (feature branch → PR → merge to main)
```

**What devs own:**
```
projects/boutique-microservices/
├── frontend/                    # React app + Dockerfile
├── backend/services/
│   ├── auth/                    # Source code + Dockerfile
│   ├── gateway/                 # Source code + Dockerfile
│   ├── orders/                  # Source code + Dockerfile
│   ├── order-service/           # Source code + Dockerfile
│   ├── product-service/         # Source code + Dockerfile
│   └── user-service/            # Source code + Dockerfile
├── docker-compose.yml           # Local dev environment
└── database/
    └── boutique_full.sql        # Schema + seed data
```

---

### Phase 2: DevOps Engineer (Platform Team)

**Step 1: Infrastructure (Terraform)**
```
Write Terraform modules:
├── modules/vpc/          # VPC, subnets, IGW, route tables
├── modules/eks/          # EKS cluster, node group, IAM roles, OIDC, EBS CSI
├── modules/ecr/          # One ECR repo per service
├── modules/argocd/       # ArgoCD + Prometheus/Grafana via Helm
├── main.tf               # Wires all modules together
├── variables.tf          # Input variables
├── terraform.tfvars      # Values (region, instance type, repo names)
└── provider.tf           # AWS provider config
```

**Step 2: CI Pipeline (GitHub Actions)**
```
Write .github/workflows/ci.yml:
├── Job 1: build-and-push
│   ├── Checkout code
│   ├── Configure AWS credentials
│   ├── Login to ECR
│   ├── Build Docker image (tag = git SHA)
│   └── Push to ECR
└── Job 2: update-manifests
    ├── sed to update image tags in gitops/k8s/*.yml
    └── Commit + push updated manifests back to GitHub
```

**Step 3: Kubernetes Manifests (GitOps folder)**
```
Write gitops/:
├── argo-cd.yml              # ArgoCD Application (what repo + path to watch)
├── kustomization.yml        # Kustomize entry point (lists all resources)
├── namespace.yml            # boutique namespace
├── secrets.yml              # DB connection strings
└── k8s/
    ├── database/
    │   ├── statefulset.yml  # PostgreSQL pod + PVC
    │   ├── service.yml      # DB service (ClusterIP)
    │   └── restore-job.yml  # Job to initialize databases on first deploy
    ├── backend/
    │   ├── auth.yml         # Deployment + Service
    │   ├── gateway.yml      # Deployment + Service
    │   ├── orders.yml       # Deployment + Service
    │   ├── order-service.yml
    │   ├── product-service.yml
    │   ├── user-service.yml
    │   └── service-monitor.yml  # Prometheus scraping
    └── frontend/
        └── deployment.yml   # Deployment + Service
```

**Step 4: Deploy**
```
1. terraform apply                          # Creates VPC, EKS, ECR, ArgoCD, Prometheus
2. aws eks update-kubeconfig                # Connect kubectl to cluster
3. Push code to trigger CI                  # Builds images, updates manifests on GitHub
4. kubectl apply -f gitops/argo-cd.yml      # Register app in ArgoCD
5. Port-forward to ArgoCD, connect repo     # One-time UI setup
6. Sync                                     # ArgoCD deploys everything
7. Restore job runs automatically           # Creates databases
8. All services healthy ✅
```

---

## What We Actually Did (Correct Order)

### One-Time Setup

| # | Action | Who | Command/Tool |
|---|--------|-----|-------------|
| 1 | Devs wrote microservices + Dockerfiles | Developers | Code editor |
| 2 | Devs tested locally with docker-compose | Developers | `docker-compose up` |
| 3 | DevOps wrote Terraform (VPC, EKS, ECR, ArgoCD) | DevOps | VS Code |
| 4 | DevOps wrote CI pipeline | DevOps | `.github/workflows/ci.yml` |
| 5 | DevOps wrote K8s manifests + Kustomization | DevOps | `gitops/` folder |
| 6 | DevOps ran `terraform apply` | DevOps | Terminal (VS Code) |
| 7 | DevOps configured GitHub Secrets (AWS creds) | DevOps | GitHub Settings |
| 8 | DevOps pushed to trigger CI (build + push images to ECR) | DevOps | `git push` |
| 9 | DevOps applied ArgoCD application | DevOps | `kubectl apply -f gitops/argo-cd.yml` |
| 10 | DevOps port-forwarded to ArgoCD | DevOps | `kubectl port-forward svc/argocd-server -n argocd 8080:80` |
| 11 | DevOps connected repo in ArgoCD UI | DevOps | ArgoCD Settings → Repos |
| 12 | DevOps synced the application | DevOps | ArgoCD UI → Sync |
| 13 | Restore job created databases automatically | Automated | K8s Job |
| 14 | All pods running ✅ | Automated | CrashLoopBackOff → Recovery |

### Every Future Code Push (Automated)

| # | What happens | Who triggers | What does the work |
|---|-------------|-------------|-------------------|
| 1 | Dev pushes code to main | Developer | `git push` |
| 2 | CI builds new images | Automated | GitHub Actions |
| 3 | CI pushes to ECR (new SHA tag) | Automated | GitHub Actions |
| 4 | CI updates image tags in `gitops/k8s/*.yml` | Automated | GitHub Actions (sed + commit) |
| 5 | ArgoCD detects manifest change on GitHub | Automated | ArgoCD (polls every 3 min) |
| 6 | ArgoCD syncs: rolls out new pods | Automated | ArgoCD (auto-sync enabled) |
| 7 | New pods pull updated images from ECR | Automated | Kubernetes |
| 8 | Zero-downtime deployment complete ✅ | Automated | Rolling update |

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
          → updates the latest image tag in github repo under path: gitops/k8s/backend & frontend(second stage)
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

---

## Order of Operations: Who Does What and When

### Phase 1: Developers Build the Application

| Step | Who | What They Do |
|------|-----|--------------|
| 1 | Dev | Write application code (Node.js/TypeScript microservices: auth, gateway, orders, order-service, product-service, user-service, frontend) |
| 2 | Dev | Write `Dockerfile` for each service |
| 3 | Dev | Create `docker-compose.yml` to run all services + PostgreSQL locally |
| 4 | Dev | Test locally: `docker compose up` — verify services talk to each other, DB migrations run, frontend renders |
| 5 | Dev | Write database migrations inside each service (`CREATE TABLE IF NOT EXISTS` on startup) |
| 6 | Dev | Push working code to GitHub (`projects/boutique-microservices/`) |

**Dev's deliverables:**
- Working source code per service
- Dockerfile per service
- docker-compose.yml (for local dev/testing)
- DB migration logic inside each service

---

### Phase 2: DevOps Engineers Build the Platform

| Step | Who | What They Do |
|------|-----|--------------|
| 7 | DevOps | Write Terraform modules (`projects/Infrastructure/`) — VPC, EKS, ECR, IAM, ArgoCD, Prometheus |
| 8 | DevOps | Write Kubernetes manifests (`gitops/k8s/`) — Deployments, Services, StatefulSet, Secrets, ConfigMaps |
| 9 | DevOps | Write `kustomization.yml` (`gitops/`) — ties all manifests together, defines image transformers |
| 10 | DevOps | Write ArgoCD Application manifest (`gitops/argo-cd.yml`) — points to gitops/ path, enables auto-sync |
| 11 | DevOps | Write CI pipeline (`.github/workflows/ci.yml`) — builds images, pushes to ECR, updates image tags |
| 12 | DevOps | Write database restore job / init container (`gitops/k8s/database/`) — ensures DBs are created on first deploy |
| 13 | DevOps | Write monitoring config (`prometheus/prometheus.yml`, `grafana/provisioning/`) |

**DevOps deliverables:**
- Terraform code (entire infrastructure)
- Kubernetes manifests (all workloads)
- CI/CD pipeline (GitHub Actions)
- GitOps config (ArgoCD Application + kustomization)
- Observability stack config

---

### Phase 3: First Deployment (One-Time)

| Step | Who | What They Do |
|------|-----|--------------|
| 14 | DevOps | Run `terraform apply` → creates VPC, EKS, ECR repos, installs ArgoCD + Prometheus via Helm |
| 15 | DevOps | Apply ArgoCD Application: `kubectl apply -f gitops/argo-cd.yml` (or Terraform does this automatically) |
| 16 | DevOps | Connect repo to ArgoCD (via Terraform secret or manual one-time setup) |
| 17 | Auto | ArgoCD syncs `gitops/` folder → creates namespace, deploys PostgreSQL StatefulSet, runs restore job, deploys all services |
| 18 | Auto | PostgreSQL starts → init container/restore job creates databases (auth_db, orders_db, products_db, users_db) |
| 19 | Auto | Microservice pods start → each runs `CREATE TABLE IF NOT EXISTS` → tables ready |
| 20 | Auto | All pods healthy ✅ |

---

### Phase 4: Ongoing Development (Every Feature/Fix)

| Step | Who | What They Do |
|------|-----|--------------|
| 21 | Dev | Makes code change, tests locally with `docker compose up` |
| 22 | Dev | Pushes to `main` branch |
| 23 | Auto | GitHub Actions triggers → builds Docker image (tag = git SHA) → pushes to ECR |
| 24 | Auto | CI updates image tag in manifests on GitHub (or ArgoCD Image Updater detects new tag) |
| 25 | Auto | ArgoCD detects manifest change → syncs → rolling update → new pods deploy |
| 26 | Auto | Zero-downtime rollout complete ✅ |

---

### Visual Timeline

```
WEEK 1-2: Devs write code
┌─────────────────────────────────────────────────┐
│ Dev: code → Dockerfile → docker-compose → test  │
└─────────────────────────────────────────────────┘

WEEK 2-3: DevOps builds platform
┌─────────────────────────────────────────────────┐
│ DevOps: Terraform → K8s manifests → CI pipeline │
│         → ArgoCD config → monitoring            │
└─────────────────────────────────────────────────┘

WEEK 3: First deploy
┌─────────────────────────────────────────────────┐
│ DevOps: terraform apply                         │
│ DevOps: kubectl apply -f gitops/argo-cd.yml     │
│ Auto:   ArgoCD syncs → pods deploy → DB inits   │
│ Result: Production running ✅                    │
└─────────────────────────────────────────────────┘

WEEK 4+: Continuous delivery
┌─────────────────────────────────────────────────┐
│ Dev: git push                                   │
│ Auto: CI → ECR → ArgoCD → new pods → done ✅    │
└─────────────────────────────────────────────────┘
```

---

### What We Actually Did (Our Journey)

1. **Devs wrote the app** — 7 microservices + React frontend + docker-compose for local testing
2. **DevOps wrote Terraform** — EKS cluster, ECR repos, VPC, IAM roles, ArgoCD Helm release
3. **DevOps wrote K8s manifests** — Deployments, Services, StatefulSet, Secrets, restore job
4. **DevOps wrote CI pipeline** — GitHub Actions: build → push to ECR → update image tags in repo
5. **DevOps ran `terraform apply`** — infrastructure created in ~15 minutes
6. **DevOps applied ArgoCD Application** — `kubectl apply -f gitops/argo-cd.yml`
7. **ArgoCD auto-synced** — pulled manifests from GitHub, deployed everything to cluster
8. **Restore job ran** — created all 4 databases in PostgreSQL
9. **Services started** — connected to DB, ran migrations, became healthy
10. **Verified** — port-forwarded to frontend, confirmed app working end-to-end

**Lessons learned along the way:**
- Always include the restore job in `kustomization.yml` (otherwise DB stays empty)
- Don't hardcode AWS account IDs in manifests (use Kustomize image transformer)
- Pull latest from GitHub before pushing ArgoCD changes (avoid out-of-sync errors)
- Enable `syncPolicy.automated` so ArgoCD syncs without manual UI clicks
- CI should update manifests on GitHub so ArgoCD can detect changes immediately
