# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

A GitOps repository for Kubernetes clusters managed by Flux CD. Changes pushed to `main` are automatically reconciled by Flux (1-minute interval). The live repo URL is `https://github.com/yoyozbi-homelab/k8s-config`.

## Repository Layout

```
apps/          # Cluster-agnostic base definitions
clusters/
  ocr1/        # OCR1 cluster (multi-node)
  rp/          # RP cluster (single node)
    overlays/apps/   # Cluster-specific patches and secrets
```

**Two app types:**
- **Helm-based**: `repository.yaml` + `release.yaml` (e.g. `apps/cloudnative-postgres/`, `apps/jellyseerr/`)
- **Raw manifests**: subdirs by resource type (`deployments/`, `services/`, `ingresses/`, `rbac/`) (e.g. `apps/dozzle/`, `apps/vaultwarden/`)

Apps without an official Helm chart should use the **bjw-s `app-template`** chart. See `apps/vikunja/release.yaml` for a full working example. Reference values: `https://raw.githubusercontent.com/bjw-s-labs/helm-charts/refs/heads/main/charts/library/common/values.yaml`

## Secret Management (SOPS)

Secrets live at `clusters/*/overlays/apps/**/secrets/*-secret.yaml` and are encrypted with SOPS + age. Only `data`/`stringData` fields are encrypted; metadata stays readable.

```bash
# Edit an encrypted secret
sops clusters/ocr1/overlays/apps/my-app/secrets/my-secret.yaml

# Decrypt to stdout
sops -d clusters/ocr1/overlays/apps/my-app/secrets/my-secret.yaml
```

**SOPS encryption is not automatic** — always encrypt before committing. Age key aliases are defined in `.sops.yaml`: `laptop`, `ocr1`/`tiny1`/`tiny2` (OCR1 cluster), `rp` (RP cluster).

## Local Validation

```bash
# Render final manifests for an app overlay
kustomize build clusters/ocr1/overlays/apps/my-app

# Dry-run apply a single manifest
kubectl apply --dry-run=client -f file.yaml

# Validate Flux CRDs
flux check
```

## Adding a New Application

1. Create `apps/my-app/kustomization.yaml` referencing resource files
2. Create `clusters/{ocr1,rp}/overlays/apps/my-app/kustomization.yaml` with cluster patches and namespace
3. Add the app to `clusters/{cluster}/overlays/apps/kustomization.yaml`
4. For secrets: create `clusters/{cluster}/overlays/apps/my-app/secrets/*-secret.yaml` and encrypt with `sops`

## Conventions

- `.yaml` extension preferred (some legacy files use `.yml`)
- 2-space indentation throughout
- All resources must have `app.kubernetes.io/name` and `app.kubernetes.io/part-of` labels
- Each app gets its own namespace (defined in the cluster overlay)
- Renovate auto-detects image/chart versions in `apps/` and `clusters/` and opens PRs weekly
