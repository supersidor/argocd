# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

ArgoCD GitOps repository for a k3s homelab cluster. Everything is deployed via the App of Apps pattern with automated sync (`prune: true`, `selfHeal: true`) — committing and pushing to `main` is the deployment mechanism.

## Architecture

### Two-level App of Apps

```
root.yaml (root-app)
├── infra.yaml    → apps/infra/     (sync-wave 0)
├── database.yaml → apps/database/  (sync-wave 1)
└── services.yaml → apps/services/  (sync-wave 2)
```

- `root.yaml` deploys only the three layer roots (`include: '{database.yaml,infra.yaml,services.yaml}'`). Bootstrap with `kubectl apply -f root.yaml`.
- Each layer root recursively picks up every ArgoCD `Application` manifest under its `apps/<layer>/` directory. Sync waves order the layers: infra first, then database, then services.
- An Application's source is either an external Helm chart (pinned `targetRevision` version) or local manifests under `manifests/<app-name>/` in this repo.

### Adding a new application

1. Create an `Application` manifest under `apps/infra/<name>/`, `apps/database/<name>/`, or `apps/services/`.
2. If it uses local manifests, put them in `manifests/<name>/`.
3. Commit and push to `main` — ArgoCD picks it up automatically.

### Key components and how they interconnect

- **MetalLB** — LoadBalancer IPs from pool `192.168.50.241-250` (`manifests/metal-lb-config/`).
- **Ingress-NGINX** — LoadBalancer service; ingress hosts use the `.homelab` domain (e.g. `argocd.homelab`).
- **external-dns** — syncs Services/Ingresses to a Pi-hole DNS server at `192.168.50.161` (`upsert-only` policy, `noop` registry). Hostnames are declared via the `external-dns.alpha.kubernetes.io/hostname` annotation. Its Pi-hole password comes from a SealedSecret.
- **sealed-secrets** — controller in `kube-system`, named `sealed-secrets-controller`, with key renewal disabled. Secrets are committed as `SealedSecret` resources; encrypt new ones with:
  ```bash
  kubeseal --controller-name=sealed-secrets-controller --controller-namespace=kube-system \
    --format yaml < secret.yaml > sealed-secret.yaml
  ```
- **Rook-Ceph** — operator + cluster in **external mode** (`external.enable: true`): it connects to a Ceph cluster running outside Kubernetes, not one it provisions. StorageClass `ceph-rbd-existing` (pool `k3s`, `reclaimPolicy: Retain`) is defined in `manifests/rook-ceph-storage-config/`.
- **CloudNativePG (CNPG)** — operator in infra; the `pg-main` 3-instance Postgres cluster in `apps/database/cnpg/` uses `ceph-rbd-existing` storage and is exposed at `pg-main.homelab` via a LoadBalancer Service selecting the primary.
- **Kyverno** — used for mutation policies, e.g. `manifests/pg-main/kyverno.yaml` wraps the CNPG initdb Job command to skip initdb when `$PGDATA` already exists (reusing pre-existing Ceph volumes).
- **cloudflared** — Cloudflare Tunnel deployment in `cloudflare-tunnel` namespace; tunnel credentials are a SealedSecret.

### Gotchas

- The `rook-ceph` and `rook-ceph-cluster` apps load Helm values via `https://raw.githubusercontent.com/supersidor/argocd/main/manifests/rook-ceph/...` URLs — edits to those values files take effect only after being pushed to `main`, and cannot be tested from a branch without changing the URL.
- `selfHeal: true` everywhere means manual `kubectl` edits to managed resources are reverted; change the repo instead.

## Common Commands

```bash
# Bootstrap everything (one-time)
kubectl apply -f root.yaml

# Check application status
kubectl get applications -n argocd

# Manual sync of an application
kubectl patch application <app-name> -n argocd --type merge \
  -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {}}}'

# Validate manifests before committing
kubectl apply --dry-run=client -f <manifest-file>
kubectl diff -f <manifest-file>
```

## Repository Configuration

- **Git URL**: https://github.com/supersidor/argocd (GitHub — despite global instructions mentioning GitLab, this repo lives on GitHub)
- **Target Branch**: main
- **ArgoCD Namespace**: argocd
- **Default Domain**: `.homelab` (resolved by Pi-hole via external-dns)
