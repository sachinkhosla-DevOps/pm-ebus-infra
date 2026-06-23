# PM eBus ITMS — Infrastructure GitOps Repo

Single source of truth for all Kubernetes infrastructure — deployed via Argo CD on both **Dev (Minikube)** and **Production (EKS)**.

## How it works

```
You edit YAML in this repo → push to main → Argo CD detects change → applies to cluster automatically
```

No manual `kubectl apply` or `helm install` needed after initial setup.

---

## Repository Structure

```
pm-ebus-infra/
│
├── environments/
│   ├── dev/                          ← Minikube (Ubuntu server 192.168.1.31)
│   │   ├── patroni/
│   │   │   ├── namespace.yaml
│   │   │   └── patroni-cluster.yaml  (2 nodes, 256Mi, 5Gi)
│   │   └── valkey/
│   │       ├── namespace.yaml
│   │       └── values.yaml           (1 replica, 2Gi, basic auth)
│   │
│   └── prod/                         ← EKS Production
│       ├── patroni/
│       │   ├── namespace.yaml
│       │   ├── patroni-cluster.yaml  (3 nodes, 16Gi, 500Gi, WAL-G backup, TLS, NLB)
│       │   ├── walg-config.yaml      (nightly S3 backup CronJob)
│       │   └── tls.yaml              (cert-manager TLS certificate)
│       ├── valkey/
│       │   ├── namespace.yaml
│       │   ├── values.yaml           (3 nodes, Sentinel, TLS, Prometheus metrics)
│       │   └── tls.yaml              (cert-manager TLS certificate)
│       └── pgbouncer/
│           └── pgbouncer.yaml        (connection pooler, AWS NLB, HPA)
│
└── argo-apps/
    ├── dev/
    │   ├── patroni-dev-app.yaml      ← points to Minikube
    │   └── valkey-dev-app.yaml
    └── prod/
        ├── patroni-prod-app.yaml     ← points to EKS (update EKS endpoint first)
        ├── valkey-prod-app.yaml
        └── pgbouncer-prod-app.yaml
```

---

## Dev vs Prod — Key Differences

| Feature | Dev (Minikube) | Prod (EKS) |
|---|---|---|
| Patroni nodes | 2 | 3 |
| Patroni memory | 256Mi | 16Gi |
| Patroni storage | 5Gi | 500Gi gp3 |
| Backup | None | WAL-G → S3 nightly |
| TLS | None | cert-manager auto-TLS |
| Valkey replicas | 1 | 2 + Sentinel auto-failover |
| Valkey auth | Plaintext in values | Kubernetes Secret |
| Connection pooler | None | PgBouncer (2 pods, HPA) |
| Monitoring | None | Prometheus + Grafana alerts |
| Load balancer | NodePort | AWS NLB (internal) |
| Node placement | Any | Dedicated DB/cache node groups |

---

## To deploy on EKS (Production)

### Step 1 — Register EKS cluster with Argo CD
```bash
# Install argocd CLI
brew install argocd

# Login to Argo CD
argocd login 192.168.1.31:8090 --username admin --insecure

# Add EKS cluster (run on a machine with EKS kubeconfig)
argocd cluster add pm-ebus-prod-cluster --name prod-eks
```

### Step 2 — Update EKS endpoint in prod app manifests
Edit `argo-apps/prod/patroni-prod-app.yaml` and `valkey-prod-app.yaml`:
```yaml
destination:
  server: https://YOUR-ACTUAL-EKS-ENDPOINT.gr7.ap-south-1.eks.amazonaws.com
```

### Step 3 — Create Kubernetes secrets on EKS (before syncing)
```bash
# Valkey password (use a strong random password in prod)
kubectl create secret generic valkey-auth \
  --from-literal=password='STRONG_RANDOM_PASSWORD_HERE' \
  -n valkey

# AWS credentials for WAL-G S3 backups
kubectl create secret generic aws-credentials \
  --from-literal=AWS_ACCESS_KEY_ID='YOUR_KEY' \
  --from-literal=AWS_SECRET_ACCESS_KEY='YOUR_SECRET' \
  -n patroni
```

### Step 4 — Apply prod Argo CD apps
```bash
kubectl apply -f argo-apps/prod/ -n argocd
```

Argo CD will deploy everything to EKS automatically.

---

## Adding a new tool (e.g. Kafka)

1. Create `environments/dev/kafka/` with Helm values for dev
2. Create `environments/prod/kafka/` with production-grade values
3. Create `argo-apps/dev/kafka-dev-app.yaml` and `argo-apps/prod/kafka-prod-app.yaml`
4. Push to main — Argo CD deploys to dev automatically, prod on manual sync

---

## Argo CD UI

| Environment | URL | Login |
|---|---|---|
| Dev (Minikube) | http://192.168.1.31:8090 | admin / (get from secret) |
| Prod (EKS) | Deploy Argo CD on EKS separately | — |
