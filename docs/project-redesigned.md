# Project: Boutique Microservices on EKS

> Full DevOps/GitOps workflow — automated CI/CD after one-time setup.

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
GitHub Actions (CI) → ECR (images) → ArgoCD (CD)
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
          → updates image tags in gitops/k8s/ manifests (sed)
          → commits + pushes updated manifests to GitHub
      → ArgoCD (running in-cluster, polls every 3 min):
          → detects manifest change on GitHub
          → syncs new state to cluster
              → new pods pull updated images from ECR
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
│  ├── kube-prometheus-stack (Prometheus + Grafana)           │
│  └── (ArgoCD Application created via kubectl after)        │
└─────────────────────────────────────────────────────────────┘
```

---

## Key Design Decisions (No Manual Intervention)

### 1. No Hardcoded Account IDs as Placeholders

**Problem we had:** Manifests contained `<AWS_ACCOUNT_ID>` placeholder — broke everything because nobody replaced it before deploying.

**Fix:** DevOps writes the full ECR image path directly in manifests (one-time activity). CI then only updates the tag portion on each build.

```yaml
# gitops/k8s/backend/auth.yml — DevOps writes this once
containers:
  - name: auth
    image: 417007889303.dkr.ecr.us-east-1.amazonaws.com/auth:abc123def
```

CI updates just the tag:
```bash
sed -i "s|image: .*/auth:.*|image: 417007889303.dkr.ecr.us-east-1.amazonaws.com/auth:${GITHUB_SHA}|" gitops/k8s/backend/auth.yml
```

**Why not Kustomize image transformer?** Overkill for this use case. Writing the full ECR path is a one-time activity when DevOps creates the manifests. After that, CI handles tag updates automatically.

**Result:** Manifests have the correct image from day one. No placeholders, no confusion.

---

### 2. Automated Image Tag Updates via CI

**Problem we had:** Had to manually edit image tags in manifests after every build.

**Fix:** CI pipeline (GitHub Actions) automatically updates the tag in manifests using `sed`, then commits and pushes back to GitHub. ArgoCD detects the manifest change and syncs.

```yaml
# In .github/workflows/ci.yml — update-manifests job
- name: Update image tags
  run: |
    sed -i "s|image: .*/auth:.*|image: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.us-east-1.amazonaws.com/auth:${{ github.sha }}|" gitops/k8s/backend/auth.yml
    # ... repeat for each service

- name: Commit and push
  run: |
    git add gitops/
    git commit -m "ci: update image tags to ${{ github.sha }}"
    git push
```

**Result:** Dev pushes code → CI builds image → CI updates tag in manifests → ArgoCD syncs automatically. No manual intervention.

---

### 3. No Manual Database Initialization

**Problem we had:** PostgreSQL started empty. Services crashed with "database does not exist". Had to manually exec into the pod and run `CREATE DATABASE`.

**Fix:** A Kubernetes Job (`restore-job.yml`) included in `kustomization.yml`. When ArgoCD syncs, it runs the job which creates all databases and loads the schema/seed data.

```yaml
# gitops/k8s/database/restore-job.yml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-restore
  namespace: boutique
spec:
  template:
    spec:
      containers:
        - name: restore
          image: postgres:15-alpine
          command: ["sh", "-c"]
          args:
            - |
              # Wait for PostgreSQL to be ready
              until pg_isready -h boutique-postgres -U postgres; do sleep 2; done
              # Create databases if they don't exist
              for DB in auth_db orders_db products_db users_db; do
                psql -h boutique-postgres -U postgres -tc "SELECT 1 FROM pg_database WHERE datname='$DB'" | \
                  grep -q 1 || psql -h boutique-postgres -U postgres -c "CREATE DATABASE $DB;"
              done
              # Load full schema + seed data
              psql -h boutique-postgres -U postgres < /sql/boutique_full.sql
          env:
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: boutique-secrets
                  key: POSTGRES_PASSWORD
      restartPolicy: Never
```

**Key:** This job MUST be listed in `kustomization.yml` — otherwise ArgoCD won't deploy it:
```yaml
# gitops/kustomization.yml
resources:
  - k8s/database/statefulset.yml
  - k8s/database/service.yml
  - k8s/database/restore-job.yml    # ← this was missing initially, caused empty DB
  - k8s/backend/auth.yml
  # ...
```

**Result:** ArgoCD syncs → PostgreSQL starts → restore job runs → creates all 4 databases + loads schema → microservices connect successfully.

---

### 4. ArgoCD Setup

**How we did it:**
- `terraform apply` installs ArgoCD via Helm (part of the argocd module)
- After infra is up, DevOps runs `kubectl apply -f gitops/argo-cd.yml` to create the Application
- One-time: port-forward to ArgoCD UI, connect repo (Settings → Repos → add PAT)
- Enable auto-sync so future changes are picked up automatically

```bash
# After terraform apply:
kubectl apply -f gitops/argo-cd.yml

# One-time UI setup:
kubectl port-forward svc/argocd-server -n argocd 8080:80
# → Open localhost:8080 → Settings → Repos → Connect with PAT
# → Applications → Sync
```

**The `gitops/argo-cd.yml` file:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: boutique
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<org>/devops-ai-playbook.git
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
```

**Result:** ArgoCD installed by Terraform, Application created via kubectl, repo connected one-time in UI. After that — fully automated.

---

### 5. Repo Connection (One-Time Manual Step)

**What we did:** Connected the private GitHub repo in ArgoCD UI using a Personal Access Token (PAT).

```
ArgoCD UI → Settings → Repositories → Connect Repo
  → Connection method: HTTPS
  → URL: https://github.com/<org>/devops-ai-playbook.git
  → Username: git
  → Password: <GitHub PAT>
```

**For next time (improvement):** This could be automated by having Terraform create a repo secret:
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
    password = var.github_pat
    username = "git"
    type     = "git"
  }
}
```

**Result:** Repo connected. ArgoCD can pull manifests from GitHub and sync.

---

## Our CI Pipeline (GitHub Actions)

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
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

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

          docker build -t $IMAGE:$TAG $BUILD_PATH
          docker push $IMAGE:$TAG

  update-manifests:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Update image tags in manifests
        run: |
          TAG=${{ github.sha }}
          ACCOUNT=${{ secrets.AWS_ACCOUNT_ID }}
          REGION=${{ env.AWS_REGION }}

          for SERVICE in auth gateway orders order-service product-service user-service; do
            sed -i "s|image: .*/$(echo $SERVICE):.*|image: ${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/${SERVICE}:${TAG}|" gitops/k8s/backend/${SERVICE}.yml
          done
          sed -i "s|image: .*/frontend:.*|image: ${ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/frontend:${TAG}|" gitops/k8s/frontend/deployment.yml

      - name: Commit and push
        run: |
          git config user.name "github-actions"
          git config user.email "github-actions@github.com"
          git add gitops/
          git commit -m "ci: update image tags to ${{ github.sha }}"
          git push
```

**How it works:**
1. Job 1 (`build-and-push`): Builds all 7 service images in parallel, pushes to ECR with SHA tag
2. Job 2 (`update-manifests`): Uses `sed` to update image tags in gitops manifests, commits + pushes to GitHub
3. ArgoCD detects the manifest change on GitHub (polls every 3 min) and syncs automatically

---

## Complete End-to-End Flow

```
┌──────────────────────────────────────────────────────────────┐
│ ONE-TIME SETUP                                               │
│                                                              │
│ 1. Fork repo                                                 │
│ 2. Set GitHub Secrets: AWS_ACCOUNT_ID, AWS_ACCESS_KEY_ID,    │
│    AWS_SECRET_ACCESS_KEY                                     │
│ 3. terraform apply                                           │
│    (creates VPC, EKS, ECR, ArgoCD, Prometheus)               │
│ 4. kubectl apply -f gitops/argo-cd.yml                       │
│ 5. Connect repo in ArgoCD UI (one-time PAT setup)            │
│ 6. Sync → everything deploys                                 │
│                                                              │
│ DONE.                                                        │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ EVERY CODE PUSH (fully automated)                            │
│                                                              │
│ Developer pushes code                                        │
│     → GitHub Actions builds + pushes to ECR                  │
│         → CI updates image tags in gitops/ manifests         │
│             → CI commits + pushes updated manifests          │
│                 → ArgoCD detects change (polls every 3 min)  │
│                     → ArgoCD syncs → new pods roll out       │
│                         → Zero-downtime deployment ✅         │
└──────────────────────────────────────────────────────────────┘
```

---

## Comparison: Original vs Redesigned

| Step | Original (manual pain) | What we did (automated) |
|------|----------------------|------------------------|
| Image tags in manifests | `<AWS_ACCOUNT_ID>` placeholder, manual fix | Full ECR path written once by DevOps, CI updates tag |
| CI updates manifests | Manual edit after each build | `sed` + git commit in CI pipeline (automated) |
| ArgoCD repo connection | Manual UI: Settings > Repos > PAT | Same (one-time manual step) |
| ArgoCD app creation | Manual: Create App in UI | `kubectl apply -f gitops/argo-cd.yml` |
| DB initialization | Manual exec into pod / broken restore Job | Restore job in `kustomization.yml` (runs automatically) |
| CI auth to AWS | — | Static access keys in GitHub Secrets |
| First sync | Manual: click Sync in ArgoCD UI | `syncPolicy.automated` — auto-syncs after first setup |
| Rollback | Re-run pipeline or manual kubectl | `git revert` → CI updates tags back → ArgoCD syncs |

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

## Networking & Ingress: Where Route53, ACM, WAF, ALB Fit

### Current State (What We Have Now)

```
User → kubectl port-forward → localhost:3000 → frontend pod
                            → localhost:8080 → ArgoCD UI
```

Services are only accessible via `kubectl port-forward`. No external access.

### What's Needed for Production

```
Internet → Route53 → ALB (with TLS + WAF) → Pods
```

### Components & Where They Go

| Component | What It Does | Where It's Configured |
|-----------|-------------|----------------------|
| **AWS Load Balancer Controller** | Creates ALBs from Ingress resources automatically | Terraform: Helm release + IAM role (IRSA) |
| **Ingress Resource** | Tells ALB Controller what paths route to which services | `gitops/k8s/ingress.yml` (new manifest) |
| **ACM Certificate** | TLS cert for HTTPS (*.yourdomain.com) | Terraform: `aws_acm_certificate` + DNS validation |
| **Route53 Hosted Zone** | DNS — maps `app.yourdomain.com` → ALB | Terraform: `aws_route53_zone` + `aws_route53_record` |
| **WAF v2 WebACL** | Rate limiting, geo-blocking, SQL injection protection | Terraform: `aws_wafv2_web_acl` + ALB association |

### How They Connect

```
┌─────────────────────────────────────────────────────────────────┐
│ Terraform creates:                                              │
│   1. ACM Certificate (*.boutique.example.com)                   │
│   2. Route53 Hosted Zone + A record (alias → ALB)               │
│   3. WAF WebACL                                                 │
│   4. AWS Load Balancer Controller (Helm + IRSA)                 │
│                                                                 │
│ GitOps manifests create:                                        │
│   5. Ingress resource (annotations trigger ALB creation)        │
│      → ALB Controller sees Ingress → creates ALB               │
│      → Attaches WAF, TLS cert via annotations                  │
│      → Routes: / → frontend, /api/auth → auth, etc.            │
└─────────────────────────────────────────────────────────────────┘
```

### Example Ingress Resource (`gitops/k8s/ingress.yml`)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: boutique-ingress
  namespace: boutique
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:417007889303:certificate/xxx
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:us-east-1:417007889303:regional/webacl/xxx
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
spec:
  rules:
    - host: app.boutique.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend
                port:
                  number: 80
          - path: /api/auth
            pathType: Prefix
            backend:
              service:
                name: auth-service
                port:
                  number: 3001
          - path: /api/orders
            pathType: Prefix
            backend:
              service:
                name: orders-service
                port:
                  number: 3003
          - path: /api/products
            pathType: Prefix
            backend:
              service:
                name: product-service
                port:
                  number: 3004
          - path: /api/users
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 3005
```

### Production Request Flow

```
Client browser
  → DNS lookup: app.boutique.example.com
      → Route53 returns ALB IP (Alias A record)
          → HTTPS request hits ALB (port 443, TLS terminated with ACM cert)
              → WAF inspects request (rate limit, block bad patterns)
                  → ALB routes by path:
                      /           → frontend pod (port 80)
                      /api/auth   → auth pod (port 3001)
                      /api/orders → orders pod (port 3003)
                      /api/*      → respective service pods
```

---

## Possible Enhancements

| Enhancement | What It Improves | Effort |
|-------------|-----------------|--------|
| **OIDC federation for CI** | Eliminates static AWS keys in GitHub Secrets — more secure | Medium (create IAM OIDC provider for GitHub) |
| **ArgoCD Image Updater** | Eliminates CI committing back to repo — cleaner git history | Medium (Helm install + annotations on Application) |
| **Terraform-managed ArgoCD Application** | Eliminates `kubectl apply -f argo-cd.yml` step | Low (add `kubernetes_manifest` resource) |
| **Terraform-managed repo secret** | Eliminates manual ArgoCD UI repo connection | Low (add `kubernetes_secret` resource) |
| **Kustomize image transformer** | Eliminates account-specific values in manifests | Low (but unnecessary with current sed approach) |
| **External Secrets Operator** | Pulls secrets from AWS Secrets Manager → no secrets in Git | Medium |
| **ArgoCD ApplicationSet** | Multi-env (dev/staging/prod) from one config | Medium |
| **Karpenter** | Better node autoscaling than cluster-autoscaler | Medium |
| **Fluent Bit → CloudWatch** | Centralized logging for all pods | Low (DaemonSet) |
| **RDS Aurora** | Production-grade DB (backups, multi-AZ, scaling) | High (replace StatefulSet) |
| **AWS LB Controller + Ingress** | External access without port-forward | Medium (see Networking section above) |

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
| 9 | DevOps | Write `kustomization.yml` (`gitops/`) — ties all manifests together (lists all resources) |
| 10 | DevOps | Write ArgoCD Application manifest (`gitops/argo-cd.yml`) — points to gitops/ path, enables auto-sync |
| 11 | DevOps | Write CI pipeline (`.github/workflows/ci.yml`) — builds images, pushes to ECR, updates image tags |
| 12 | DevOps | Write database restore job (`gitops/k8s/database/restore-job.yml`) — ensures DBs are created on first deploy |
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
| 15 | DevOps | Apply ArgoCD Application: `kubectl apply -f gitops/argo-cd.yml` |
| 16 | DevOps | Port-forward to ArgoCD UI, connect repo with PAT (one-time) |
| 17 | DevOps | Sync application in ArgoCD UI |
| 18 | Auto | ArgoCD syncs `gitops/` folder → creates namespace, deploys PostgreSQL StatefulSet, runs restore job, deploys all services |
| 19 | Auto | PostgreSQL starts → restore job creates databases (auth_db, orders_db, products_db, users_db) |
| 19 | Auto | Microservice pods start → each runs `CREATE TABLE IF NOT EXISTS` → tables ready |
| 20 | Auto | All pods healthy ✅ |

---

### Phase 4: Ongoing Development (Every Feature/Fix)

| Step | Who | What They Do |
|------|-----|--------------|
| 21 | Dev | Makes code change, tests locally with `docker compose up` |
| 22 | Dev | Pushes to `main` branch |
| 23 | Auto | GitHub Actions triggers → builds Docker image (tag = git SHA) → pushes to ECR |
| 24 | Auto | CI updates image tags in manifests on GitHub (sed + commit + push) |
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
6. **DevOps pushed code to GitHub** — CI triggered, built images, pushed to ECR, updated manifest tags
7. **DevOps applied ArgoCD Application** — `kubectl apply -f gitops/argo-cd.yml`
8. **DevOps port-forwarded to ArgoCD** — `kubectl port-forward svc/argocd-server -n argocd 8080:80`
9. **DevOps connected repo in ArgoCD UI** — Settings → Repos → HTTPS + PAT
10. **DevOps synced application** — Applications → boutique → Sync
11. **Restore job ran** — created all 4 databases in PostgreSQL
12. **All services started** — connected to DB, ran migrations, became healthy
13. **Verified** — port-forwarded to frontend, confirmed app working end-to-end

**Lessons learned along the way:**
- Always include the restore job in `kustomization.yml` (otherwise DB stays empty)
- Write full ECR image paths in manifests (no `<AWS_ACCOUNT_ID>` placeholders)
- Pull latest from GitHub before pushing changes (avoid out-of-sync errors)
- Enable `syncPolicy.automated` so ArgoCD syncs without manual UI clicks after first time
- CI should update manifests on GitHub so ArgoCD can detect changes immediately
