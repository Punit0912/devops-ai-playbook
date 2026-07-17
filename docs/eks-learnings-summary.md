# EKS Learnings — Complete Summary

## Table of Contents
1. [EKS Foundation — What Gets Created](#1-eks-foundation)
2. [What You Hand Over to a Microservices Team](#2-what-you-hand-over)
3. [IRSA & OIDC — Pod-Level AWS Access](#3-irsa--oidc)
4. [EBS CSI Driver](#4-ebs-csi-driver)
5. [Ingress Patterns — All Scenarios](#5-ingress-patterns)
6. [GitOps with ArgoCD](#6-gitops-with-argocd)
7. [Without GitOps — Traditional CI/CD](#7-without-gitops)
8. [Deploying Third-Party Apps (GitLab, Jira, etc.)](#8-third-party-apps)
9. [Full AWS Architecture Scenarios](#9-full-aws-architecture-scenarios)
10. [CI/CD Pipeline Patterns](#10-cicd-pipeline-patterns)

---

## 1. EKS Foundation

What `terraform apply` creates:

```
┌─────────────────────────────────────────────────────────┐
│ AWS Account                                             │
│                                                         │
│  VPC (10.1.0.0/16)                                     │
│  ├── Subnet 1 (us-east-1a)                             │
│  ├── Subnet 2 (us-east-1b)                             │
│  ├── Subnet 3 (us-east-1c)                             │
│  ├── Internet Gateway                                  │
│  └── Route Table (0.0.0.0/0 → IGW)                    │
│                                                         │
│  EKS Cluster                                           │
│  ├── Control Plane (managed by AWS)                    │
│  ├── Node Group (m7i-flex.large × 1-2)                 │
│  ├── OIDC Provider (for IRSA)                          │
│  └── EBS CSI Driver (addon)                            │
│                                                         │
│  ECR Repositories (7 repos)                            │
│                                                         │
│  IAM Roles                                             │
│  ├── eks-cluster-role (for control plane)              │
│  ├── eks-node-role (for worker nodes)                  │
│  └── ebs-csi-irsa (for EBS CSI driver)                │
│                                                         │
│  Helm Releases (on the cluster)                        │
│  ├── ArgoCD (namespace: argocd)                        │
│  └── kube-prometheus-stack (namespace: monitoring)     │
└─────────────────────────────────────────────────────────┘
```

---

## 2. What You Hand Over to a Microservices Team

As a platform/DevOps engineer, you provide:

| Component | What | Status |
|-----------|------|--------|
| EKS Cluster | Compute platform | ✅ Created via Terraform |
| Namespace | Team's isolated space | ✅ Created per team |
| Ingress Controller | NGINX/ALB pods that route traffic | ✅ DaemonSet + Service + IngressClass |
| ALB + TargetGroupBinding | External traffic reaches the controller | ✅ Targets = Ingress Controller pods |
| RBAC | Team can only access their namespace | ✅ Role + RoleBinding |
| Resource Quotas | CPU/memory limits per namespace | ✅ Prevents noisy neighbors |
| Network Policies | Control pod-to-pod communication | ✅ Isolation between namespaces |
| ECR Repos | Where their images live | ✅ Created per service |
| ArgoCD Project | Scoped GitOps access | ✅ Team can only deploy to their namespace |

**Team receives:**
- Namespace name
- ECR repo URLs
- Ingress class name
- ArgoCD project name
- Service account with IRSA (if they need AWS access)

---

## 3. IRSA & OIDC — Pod-Level AWS Access

### Problem
Pods need AWS access (S3, SQS, DynamoDB) without hardcoding credentials.

### Solution: IRSA (IAM Roles for Service Accounts)

```
Pod → K8s Service Account → OIDC Token → IAM validates → Grants IAM Role
```

### How it works:

```terraform
# 1. OIDC Provider (created once per cluster)
resource "aws_iam_openid_connect_provider" "eks" {
  url             = aws_eks_cluster.eks.identity[0].oidc[0].issuer
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.eks.certificates[0].sha1_fingerprint]
}

# 2. IAM Role with trust policy (per workload)
resource "aws_iam_role" "my_app_role" {
  assume_role_policy = jsonencode({
    Statement = [{
      Action = "sts:AssumeRoleWithWebIdentity"
      Principal = { Federated = aws_iam_openid_connect_provider.eks.arn }
      Condition = {
        StringEquals = {
          "${oidc_issuer}:sub" = "system:serviceaccount:my-namespace:my-service-account"
        }
      }
    }]
  })
}

# 3. Attach permissions
resource "aws_iam_role_policy_attachment" "s3_access" {
  role       = aws_iam_role.my_app_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}
```

```yaml
# 4. K8s Service Account (annotated with IAM role)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-app-role
---
# 5. Pod uses the service account
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: my-service-account
      containers:
        - name: app
          # AWS SDK auto-discovers credentials via IRSA
```

### When IRSA is needed:
- EBS CSI Driver → needs to create/attach EBS volumes
- ALB Controller → needs to create ALBs, target groups
- App pods → need S3, SQS, DynamoDB access
- External Secrets Operator → needs Secrets Manager access

---

## 4. EBS CSI Driver

### Why it's needed
EKS can't provision EBS volumes for PersistentVolumeClaims without this driver. It's the bridge between Kubernetes storage requests and AWS EBS API.

### What happens without it:
```
Pod requests PVC → StorageClass says "use gp2" → NOTHING HAPPENS → Pod stuck in Pending
```

### What happens with it:
```
Pod requests PVC → StorageClass "gp2" → EBS CSI Driver → AWS API: CreateVolume → EBS attached to node → Pod starts
```

### In this project:
PostgreSQL StatefulSet uses a PVC with `storageClassName: gp2` — the EBS CSI driver provisions and attaches the volume.

---

## 5. Ingress Patterns — All Scenarios

### Scenario A: ALB Controller Only (No NGINX)

**Use when:** Simple routing, need WAF/Shield natively, no custom routing logic.

```
Internet → ALB (WAF + Shield) → Target Groups → Pods directly (IP mode)
```

```yaml
# Ingress resource (creates ALB automatically)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:...
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:...
spec:
  rules:
    - host: app.company.com
      http:
        paths:
          - path: /api
            backend: { service: { name: gateway, port: { number: 3001 } } }
          - path: /
            backend: { service: { name: frontend, port: { number: 3000 } } }
```

**ALB Controller pod** watches this and calls AWS API to create:
- ALB
- Listener rules (host + path → target group)
- Target groups (registered with pod IPs)

**Pros:** Simple, native AWS integration, WAF/Shield via annotations
**Cons:** Limited routing features, no rate-limiting, no custom headers

---

### Scenario B: NGINX Controller + NLB (No ALB Controller needed)

**Use when:** Need advanced routing (regex, rewrites, rate-limiting), don't need WAF.

```
Internet → NLB → NGINX Controller Pods → ClusterIP Services → App Pods
```

```yaml
# NGINX Controller Service (creates NLB)
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  type: LoadBalancer
  ports:
    - port: 80
    - port: 443
---
# Ingress resource (NGINX reads this)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/rate-limit: "10"
    nginx.ingress.kubernetes.io/rewrite-target: /$1
spec:
  rules:
    - host: app.company.com
      http:
        paths:
          - path: /api/(.*)
            backend: { service: { name: gateway, port: { number: 3001 } } }
```

**Pros:** Powerful routing, regex, rate-limiting, custom headers, request transforms
**Cons:** No native WAF integration (need separate WAF on NLB or CloudFront)

---

### Scenario C: ALB (Terraform/Manual) + NGINX + TargetGroupBinding (Hybrid)

**Use when:** Need WAF + Shield + advanced routing. Production-grade.

```
Internet → ALB (WAF + Shield + Cognito) → Target Group → NGINX Pods → ClusterIP → App Pods
```

Step 1: Create ALB via Terraform (not via controller)
```terraform
resource "aws_lb" "main" {
  name               = "eks-main-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids
}

resource "aws_wafv2_web_acl_association" "main" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}

resource "aws_lb_target_group" "nginx" {
  name        = "nginx-tg"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    path = "/healthz"
  }
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = var.acm_cert_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nginx.arn
  }
}
```

Step 2: NGINX Controller with ClusterIP (not LoadBalancer)
```yaml
# NGINX Service — NO LoadBalancer, just ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
spec:
  type: ClusterIP
  ports:
    - port: 80
```

Step 3: TargetGroupBinding (glue between ALB and NGINX)
```yaml
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: nginx-tgb
spec:
  serviceRef:
    name: ingress-nginx-controller
    port: 80
  targetGroupARN: arn:aws:elasticloadbalancing:...:targetgroup/nginx-tg/abc123
```

**Pros:** Full WAF control, Cognito auth, Shield, custom rules — all on the ALB. NGINX handles routing logic.
**Cons:** More complex setup, two layers of config.

---

### Scenario D: CloudFront + ALB + NGINX (Global CDN)

**Use when:** Global users, need edge caching, DDoS protection at edge.

```
Users → CloudFront (WAF at edge) → ALB (origin) → NGINX → Services
```

---

### When to use which:

| Requirement | Pattern |
|-------------|---------|
| Simple app, need WAF | **A** (ALB Controller only) |
| Complex routing, no WAF needed | **B** (NGINX + NLB) |
| WAF + complex routing | **C** (ALB Terraform + NGINX + TGB) |
| Global CDN + WAF + routing | **D** (CloudFront + ALB + NGINX) |
| WAF rule groups (not supported by controller) | **C** or **D** (must create ALB separately) |

---

## 6. GitOps with ArgoCD

### Setup Steps

1. `terraform apply` → installs ArgoCD on cluster
2. `kubectl apply -f gitops/argo-cd.yml` → registers your app
3. ArgoCD watches Git → auto-deploys on changes

### How it works:

```
Developer pushes code
        │
        ▼
GitHub Actions (CI)
├── Builds Docker images
├── Pushes to ECR
├── Updates image tags in gitops/k8s/*.yml
└── Commits updated manifests back to repo
        │
        ▼
ArgoCD (CD)
├── Detects manifest change in gitops/
├── Compares desired state (Git) vs live state (cluster)
└── Syncs: applies new manifests to EKS
        │
        ▼
New pods running with updated images
```

### Key concepts:

| Concept | Meaning |
|---------|---------|
| **Synced** | Git state = Cluster state |
| **OutOfSync** | Someone changed something (Git or cluster) |
| **Self-Heal** | If someone manually changes cluster, ArgoCD reverts it |
| **Auto-Sync** | ArgoCD applies changes without manual trigger |
| **Prune** | Delete resources from cluster that were removed from Git |

### ArgoCD Application manifest:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: boutique
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/Punit0912/devops-ai-playbook.git'
    targetRevision: main
    path: gitops                    # ← Kustomize reads from here
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: boutique
  syncPolicy:
    automated:                      # ← enables auto-sync
      prune: true                   # ← deletes removed resources
      selfHeal: true                # ← reverts manual cluster changes
```

### Accessing ArgoCD:
```bash
# Port-forward (our current setup — no ingress controller)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

---

## 7. Without GitOps — Traditional CI/CD

### How it differs:

| | With GitOps (ArgoCD) | Without GitOps |
|--|-----|------|
| Who deploys? | ArgoCD pulls from Git | CI/CD pipeline pushes to cluster |
| Source of truth | Git repo (manifests) | Whatever was last applied |
| Drift detection | ArgoCD detects and alerts/fixes | None — manual kubectl can diverge |
| Rollback | `git revert` → ArgoCD syncs | `kubectl rollout undo` or re-run pipeline |
| Audit trail | Git commits | CI/CD logs (harder to trace) |

### Traditional pipeline (push-based):
```yaml
# GitHub Actions — deploys directly to EKS
- name: Deploy to EKS
  run: |
    aws eks update-kubeconfig --name eks-cluster --region us-east-1
    kubectl set image deployment/frontend frontend=$IMAGE_URI -n boutique
    kubectl rollout status deployment/frontend -n boutique
```

### Problems with push-based:
- CI/CD needs cluster credentials (security risk)
- No drift detection
- Manual `kubectl apply` can break things silently
- Rollback is manual

---

## 8. Third-Party Apps (GitLab, Jira, Grafana, etc.)

### Pattern: Helm chart from vendor

These companies provide Helm charts. You don't build images — you consume theirs.

#### With GitOps (ArgoCD):
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitlab
  namespace: argocd
spec:
  source:
    repoURL: https://charts.gitlab.io/
    chart: gitlab
    targetRevision: 7.11.0
    helm:
      values: |
        global:
          hosts:
            domain: company.com
          ingress:
            class: nginx
            annotations:
              cert-manager.io/cluster-issuer: letsencrypt
  destination:
    server: https://kubernetes.default.svc
    namespace: gitlab
```

#### Without GitOps (Helm CLI):
```bash
helm repo add gitlab https://charts.gitlab.io/
helm install gitlab gitlab/gitlab \
  --namespace gitlab \
  --set global.hosts.domain=company.com \
  --set global.ingress.class=nginx
```

#### Via Terraform (declarative, but not GitOps):
```terraform
resource "helm_release" "gitlab" {
  name       = "gitlab"
  namespace  = "gitlab"
  repository = "https://charts.gitlab.io/"
  chart      = "gitlab"
  version    = "7.11.0"

  values = [
    yamlencode({
      global = {
        hosts = { domain = "company.com" }
      }
    })
  ]
}
```

### Comparison for third-party apps:

| Method | Pros | Cons |
|--------|------|------|
| ArgoCD + Helm source | GitOps, drift detection, auto-upgrade | Need ArgoCD running |
| `helm install` | Quick, simple | No audit trail, drift possible |
| Terraform `helm_release` | Declarative, versioned | Terraform state dependency, slow |

---

## 9. Full AWS Architecture Scenarios

### Scenario 1: Simple Startup (this project)

```
Developer → GitHub → Actions (build + push ECR)
                                    │
                                    ▼ (updates manifests)
                              ArgoCD (on EKS)
                                    │
                                    ▼
                              EKS Cluster
                              ├── frontend (ClusterIP)
                              ├── gateway (ClusterIP)
                              ├── auth (ClusterIP)
                              ├── orders (ClusterIP)
                              ├── products (ClusterIP)
                              └── postgres (StatefulSet + EBS)

Access: kubectl port-forward (dev only)
Cost: ~$75/month (EKS + 1 node)
```

---

### Scenario 2: Production with ALB Controller

```
Route53: app.company.com → ALB
                              │
                    ┌─────────┴─────────┐
                    │    AWS ALB         │
                    │  (created by       │
                    │   ALB Controller)  │
                    │  + ACM cert (TLS)  │
                    └─────────┬─────────┘
                              │
                    EKS Cluster
                    ├── ALB Controller (watches Ingress)
                    ├── frontend → /
                    ├── gateway → /api/*
                    └── postgres (StatefulSet + EBS)

ArgoCD: manages all deployments
Monitoring: Prometheus + Grafana (port-forward or internal Ingress)
Cost: ~$100/month (EKS + node + ALB)
```

---

### Scenario 3: Enterprise with WAF + Cognito + NGINX

```
Route53: *.company.com → CloudFront (optional)
                              │
                    ┌─────────┴─────────┐
                    │    AWS ALB         │
                    │  (Terraform)       │
                    │  + WAF v2          │
                    │  + Shield Advanced │
                    │  + Cognito Auth    │
                    └─────────┬─────────┘
                              │
                    ┌─────────┴─────────┐
                    │  NGINX Controller  │
                    │  (TargetGroupBind) │
                    │  rate-limit        │
                    │  rewrites          │
                    │  custom headers    │
                    └─────────┬─────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
         Namespace A     Namespace B     Namespace C
         (Team Alpha)    (Team Beta)     (Platform)
         ├── svc-1       ├── svc-1       ├── ArgoCD
         ├── svc-2       ├── svc-2       ├── Grafana
         └── svc-3       └── svc-3       └── Prometheus

CI/CD: GitHub Actions → ECR → ArgoCD (per team)
DNS: Route53 hosted zone with alias records
TLS: ACM certificates (auto-renewed)
Auth: Cognito user pools (login before reaching app)
Cost: ~$500+/month
```

---

### Scenario 4: Multi-Cluster (DR / Multi-Region)

```
Route53 (Latency-based routing)
├── us-east-1: ALB → EKS Cluster A
└── eu-west-1: ALB → EKS Cluster B

ArgoCD ApplicationSet:
  - Deploys same app to both clusters
  - Git is single source of truth

Database:
  - RDS with read replicas (cross-region)
  - Or Aurora Global Database
```

---

## 10. CI/CD Pipeline Patterns

### Pattern A: Full GitOps (This Project)

```yaml
# CI only — no kubectl in pipeline
jobs:
  build:
    steps:
      - Build Docker images
      - Push to ECR
      - Update image tags in gitops/ manifests (sed)
      - Git commit + push updated manifests

# CD handled by ArgoCD (separate)
# ArgoCD detects commit → syncs to cluster
```

**Pipeline has NO cluster access.** ArgoCD pulls changes.

---

### Pattern B: Push-based (Traditional)

```yaml
jobs:
  build-and-deploy:
    steps:
      - Build Docker images
      - Push to ECR
      - aws eks update-kubeconfig
      - kubectl apply -f k8s/
      - kubectl rollout status
```

**Pipeline HAS cluster access.** Pushes directly.

---

### Pattern C: Hybrid (CI pushes, ArgoCD watches)

```yaml
jobs:
  build:
    steps:
      - Build + Push to ECR
  
  update-manifests:
    steps:
      - Update image tags in Git
      - Push to a separate "deploy" branch or repo
      
# ArgoCD watches the deploy branch/repo
```

**Separation:** App code repo ≠ Deployment config repo (recommended at scale).

---

### Pattern D: With Preview Environments

```yaml
jobs:
  build:
    steps:
      - Build + Push to ECR
      
  preview:
    if: github.event_name == 'pull_request'
    steps:
      - Create ArgoCD Application for PR
      - Deploy to preview namespace
      - Comment PR with preview URL
      
  cleanup:
    if: github.event.action == 'closed'
    steps:
      - Delete preview namespace
      - Delete ArgoCD Application
```

---

## Quick Reference: AWS Services in EKS Context

| AWS Service | Role in EKS Architecture |
|-------------|--------------------------|
| **Route53** | DNS — maps domain names to ALB/NLB IPs |
| **ACM** | TLS certificates (free, auto-renewed) for ALB/CloudFront |
| **ALB** | Layer 7 load balancer — HTTP routing, WAF integration |
| **NLB** | Layer 4 load balancer — TCP/UDP, used with NGINX controller |
| **WAF v2** | Web Application Firewall — SQL injection, XSS, rate limiting, IP blocking |
| **Shield** | DDoS protection (Standard = free, Advanced = paid) |
| **Cognito** | User authentication — add login before reaching your app |
| **CloudFront** | CDN — edge caching, global distribution, WAF at edge |
| **ECR** | Container image registry |
| **EBS** | Block storage for StatefulSets (databases) |
| **EFS** | Shared file storage (multiple pods, ReadWriteMany) |
| **Secrets Manager** | Store secrets, synced to K8s via External Secrets Operator |
| **CloudWatch** | Logs (via Fluent Bit) and metrics forwarding |
| **CodePipeline** | AWS-native CI/CD (alternative to GitHub Actions) |

---

## Decision Matrix

| Question | If Yes → | If No → |
|----------|----------|---------|
| Need WAF rule groups? | Create ALB via Terraform | ALB Controller can handle it |
| Need Cognito auth? | ALB via Terraform (Cognito action on listener) | ALB Controller |
| Need regex routing? | NGINX Controller | ALB Controller |
| Need rate limiting per endpoint? | NGINX Controller | ALB is basic |
| Single team, simple app? | ALB Controller only | — |
| Multi-team, shared infra? | ALB (Terraform) + NGINX + TGB | — |
| Need drift detection? | ArgoCD (GitOps) | — |
| Need fast rollback? | ArgoCD (`git revert`) | `kubectl rollout undo` |
| Third-party app (Helm chart)? | ArgoCD Helm source OR `helm install` | — |

---

*Generated from devops-ai-playbook learning project*
