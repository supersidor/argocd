# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is an ArgoCD GitOps repository for managing Kubernetes applications and infrastructure components. The repository follows the App of Apps pattern where a root ArgoCD application manages child applications.

## Architecture

### Directory Structure
- `root.yaml` - Root ArgoCD application that manages all apps in the `apps/` directory
- `apps/` - Contains ArgoCD Application manifests organized by:
  - `infra/` - Infrastructure components (MetalLB, Ingress-NGINX, ArgoCD config)
  - `services/` - Service applications
- `manifests/` - Raw Kubernetes manifests deployed by ArgoCD applications

### Key Components

1. **ArgoCD** - GitOps continuous delivery tool managing all deployments
2. **MetalLB** - Load balancer for bare-metal Kubernetes (provides external IPs)
3. **Ingress-NGINX** - Ingress controller for HTTP/HTTPS routing

### Deployment Pattern

Applications use the following pattern:
- ArgoCD Application manifest in `apps/` references either:
  - Helm charts from external repositories
  - Local manifests in `manifests/` directory
- All applications sync automatically with `prune: true` and `selfHeal: true`
- Sync waves control deployment order (e.g., MetalLB deploys before other services)

## Common Commands

### ArgoCD Operations
```bash
# Apply root application (bootstraps everything)
kubectl apply -f infra.yaml

# Check application status
kubectl get applications -n argocd

# Manual sync of an application
kubectl patch application <app-name> -n argocd --type merge -p '{"operation": {"initiatedBy": {"username": "admin"}, "sync": {}}}'
```

### Adding New Applications
1. Create application manifest in `apps/infra/` or `apps/services/`
2. If using local manifests, place them in `manifests/<app-name>/`
3. Commit and push - ArgoCD will auto-detect and deploy

### Testing Changes
```bash
# Dry-run to see what would be applied
kubectl diff -f <manifest-file>

# Validate YAML syntax
kubectl apply --dry-run=client -f <manifest-file>
```

## Repository Configuration

- **Git URL**: https://github.com/supersidor/argocd
- **Target Branch**: main
- **ArgoCD Namespace**: argocd
- **Default Domain**: homelab (used in ingress hosts)