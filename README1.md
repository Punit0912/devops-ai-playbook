# Boutique Microservices on EKS — Simple Guide (README1)

This is a **plain-English walkthrough** of what we are building, why, and the exact
steps we take. It also has small **"✅ Check now"** boxes so you can confirm each
part is healthy before moving on.

> Audience: someone new to Kubernetes / AWS / GitOps. No prior deep knowledge needed.

---

## 1. What are we trying to achieve?

We want to run a small online shop ("Boutique") made of **many small services**
(frontend, gateway, auth, orders, products, users, and a database) on the cloud,
in a way that is **automated and repeatable**.

In simple words:

1. We write code + configuration.
2. We push to GitHub.
3. A robot pipeline builds the app into images and stores them.
4. Another robot (ArgoCD) automatically deploys those images to the cluster.
5. We can watch everything with dashboards (Grafana/Prometheus).

The big idea is **GitOps**: *Git is the single source of truth.* Whatever is in the
Git repo is what should be running in the cluster. If they differ, ArgoCD fixes it.

---

## 2. The pieces (in kid-simple terms)

| Piece | What it is | Think of it as |
|-------|-----------|----------------|
| **AWS** | The cloud provider | The land we rent |
| **VPC** | Private network | The fenced yard |
| **EKS** | Managed Kubernetes | The robot that runs our apps |
| **ECR** | Image storage | A warehouse for app "boxes" (images) |
| **Terraform** | Infra-as-code | The blueprint that builds the yard + robot |
| **GitHub Actions** | CI pipeline | The factory that builds the boxes |
| **ArgoCD** | GitOps deployer | The delivery robot that installs boxes |
| **Prometheus/Grafana** | Monitoring | The CCTV + dashboards |
| **Postgres** | Database | The shop's filing cabinet |

---

## 3. The overall flow

```
  You (code)  ──push──▶  GitHub
                            │
                            ▼
                  GitHub Actions (CI)
             build images ──▶ push to ECR
             update image tags in gitops/k8s/*.yml
                            │
                            ▼
                  ArgoCD (watches Git)
             sees new manifests ──▶ deploys to EKS
                            │
                            ▼
        EKS runs: frontend + backend + Postgres
                            │
                            ▼
        Prometheus/Grafana watch it all
```

---

## 4. Step-by-step, with checks

### Step 0 — Prerequisites
- `kubectl`, `aws` CLI, `terraform`, and `git` installed.
- AWS credentials configured.
- Access to the cluster (`aws eks update-kubeconfig ...`).

> **Note (VDI):** This machine has **no local Docker**, so we do **not** build
> images locally. GitHub Actions builds them for us. Terraform also avoids the
> Kubernetes provider (it crashed on the VDI) and uses **Helm-only** for add-ons.

---

### Step 1 — Build the infrastructure (Terraform)

We create the network (VPC), the Kubernetes cluster (EKS), the image warehouse
(ECR), and install ArgoCD + monitoring via Helm.

```powershell
cd projects/Infrastructure
terraform init
terraform plan
terraform apply
```

> **✅ Check infra now** — cluster reachable & nodes ready:
> ```powershell
> aws eks update-kubeconfig --name <cluster-name> --region <region>
> kubectl get nodes
> ```
> You should see one or more nodes with STATUS = `Ready`.

> **✅ Check add-ons now** — ArgoCD and monitoring installed:
> ```powershell
> kubectl get pods -n argocd
> kubectl get pods -n monitoring
> ```

---

### Step 2 — Build & push images (GitHub Actions)

Pushing to `main` (or running the workflow manually) triggers
`.github/workflows/ci.yml`, which:

1. Builds a Docker image for each service (auth, gateway, orders, order-service,
   product-service, user-service, frontend).
2. Pushes each image to **ECR**, tagged with the commit SHA.
3. Auto-updates the image tags inside `gitops/k8s/*` and commits them back.

> **✅ Check pipeline now** — open the **Actions** tab on GitHub and confirm the
> "Boutique CI Pipeline" run is green. Then pull the tag updates locally:
> ```powershell
> git pull --rebase
> ```

---

### Step 3 — Let ArgoCD deploy (GitOps)

ArgoCD watches the `gitops` folder in the repo and applies it to the cluster.

Register the app once:
```powershell
kubectl apply -f gitops/argo-cd.yml
```

> **✅ Check ArgoCD now** — app should be `Synced` / `Healthy`:
> ```powershell
> kubectl get applications -n argocd
> ```
> If it shows `OutOfSync`, trigger a sync (UI button, or `argocd app sync boutique`).

> **✅ Check pods now** — all app pods should be `Running`:
> ```powershell
> kubectl get pods -n boutique
> ```

---

### Step 4 — Seed the database (restore Job)

The database needs its tables and demo data. A one-shot **Job**
(`gitops/k8s/database/restore-job.yml`) waits for Postgres, restores a full dump,
and creates the `auth_db` / `orders_db` schemas.

**Important lesson learned:** Kubernetes Jobs are **immutable**. When the Job
finished and got cleaned up, ArgoCD kept seeing it as `Missing`/`OutOfSync` and
sync failed. We fixed this by turning the Job into an **ArgoCD sync hook**:

```yaml
annotations:
  argocd.argoproj.io/hook: Sync
  argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
```

Now ArgoCD deletes + recreates the Job on each sync instead of trying to patch it.

> **✅ Check database now** — the databases exist:
> ```powershell
> kubectl exec -it -n boutique statefulset/boutique-postgres -- psql -U postgres -c "\l"
> ```
> You should see `auth_db`, `orders_db`, etc.

> **✅ Check restore Job now**:
> ```powershell
> kubectl get jobs -n boutique
> kubectl logs -n boutique job/boutique-db-restore
> ```

---

### Step 5 — Open the app on your machine (port-forward)

Because services are internal (`ClusterIP`), we tunnel them to `localhost`.
Run each in its **own** terminal (it blocks while running):

```powershell
# Frontend UI  -> http://localhost:3000
kubectl port-forward -n boutique svc/frontend 3000:3000

# API gateway  -> http://localhost:3001
kubectl port-forward -n boutique svc/gateway 3001:3001
```

Optional backend services / database:
```powershell
kubectl port-forward -n boutique svc/auth 3002:3002
kubectl port-forward -n boutique svc/product-service 3003:3003
kubectl port-forward -n boutique svc/orders 3005:3005
kubectl port-forward -n boutique svc/user-service 3006:3006
kubectl port-forward -n boutique svc/boutique-postgres 5432:5432
```

> **✅ Check app now** — open <http://localhost:3000> in a browser. You should see
> the shop and be able to browse products.

---

## 5. Handy "health check" commands (cheat sheet)

```powershell
# Nodes / cluster
kubectl get nodes

# Everything in the app namespace
kubectl get all -n boutique

# Just pods (most common check)
kubectl get pods -n boutique

# Why is a pod unhappy?
kubectl describe pod <pod-name> -n boutique
kubectl logs <pod-name> -n boutique

# ArgoCD sync state
kubectl get applications -n argocd

# Monitoring stack
kubectl get pods -n monitoring
```

---

## 6. Common problems we already solved

| Symptom | Cause | Fix |
|--------|-------|-----|
| Terraform provider crash on VDI | Kubernetes provider unstable on VDI | Removed K8s provider; use **Helm-only** with `create_namespace=true` |
| Backend `CrashLoopBackOff` | DB not seeded (restore Job failed) | Restore Job with tolerant `psql ... || true` |
| Restore aborted on benign errors | `ON_ERROR_STOP=1` on a full cluster dump | Removed it; ignore "already exists" errors |
| ArgoCD `OutOfSync` / sync fails on the Job | Jobs are immutable / get garbage-collected | Made the Job an **ArgoCD sync hook** |

---

## 7. What's next — AIOps layer (coming up)

After the app is stable, we add an **AIOps** layer to detect and explain problems
automatically:

- **Fluent Bit** — ship container logs out of the cluster.
- **Prometheus LoadBalancer** — expose metrics externally.
- **Lambda functions** — fetch logs / metrics / health on demand.
- **Bedrock (LLM)** — reason over that data to summarize incidents.

We'll document each of those with the same "✅ Check now" style.

---

*Last updated: rebuild of the EKS GitOps environment (Boutique demo).*
