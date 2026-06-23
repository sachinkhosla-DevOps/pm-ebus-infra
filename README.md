# PM eBus ITMS — Infrastructure GitOps Repo

This repository is the **single source of truth** for all Kubernetes infrastructure deployed on the PM-eBus Ubuntu server (192.168.1.31) via Argo CD.

## How it works

1. You edit a YAML file in this repo (e.g. increase Patroni memory)
2. Push to `main` branch
3. Argo CD automatically detects the change and applies it to the cluster — no manual `kubectl` needed

## Structure

```
pm-ebus-infra/
├── patroni/                  # PostgreSQL HA cluster (Zalando Patroni)
│   ├── namespace.yaml
│   └── patroni-cluster.yaml
├── valkey/                   # Cache (Valkey 9.1.0)
│   ├── namespace.yaml
│   └── values.yaml
└── argo-apps/                # Argo CD Application definitions
    ├── patroni-app.yaml
    ├── valkey-app.yaml
    └── patroni-backup-workflow.yaml   # Nightly backup CronWorkflow
```

## Argo CD UI

- Local: http://192.168.1.31:8090
- Login: admin / (get from secret)

## Adding a new tool

1. Create a new folder (e.g. `kafka/`)
2. Add manifests or helm values
3. Create `argo-apps/kafka-app.yaml` pointing to that folder
4. Push to main — Argo CD deploys automatically
