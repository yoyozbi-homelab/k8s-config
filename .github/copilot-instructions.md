# Copilot Instructions for k8s-config

This is a Kubernetes configuration repository using Flux CD for GitOps deployments across multiple clusters.

## Repository Structure

- **`apps/`** - Base application definitions (Helm releases and raw Kubernetes manifests)
  - Each app subdirectory contains `kustomization.yaml` plus resources (Deployments, Services, Ingresses, etc.)
  - Helm-based apps have `repository.yaml` and `release.yaml`
  - Raw Kubernetes apps have resource files organized by type (deployments/, services/, ingresses/)

- **`clusters/`** - Cluster-specific overlays using Kustomize
  - `ocr1/` and `rp/` directories represent different Kubernetes clusters
  - Each cluster has `git-repository.yaml` (Flux GitRepository) and overlay structure
  - `overlays/apps/` contains cluster-specific customizations and secrets for each app
  - Secrets are encrypted with SOPS using age keys

- **`.sops.yaml`** - Secret encryption configuration
  - Defines age keys for different clusters and machines
  - Automatically encrypts files matching patterns in `clusters/*/.*-secret.yaml`

- **`renovate.json`** - Dependency update automation
  - Enables Flux for YAML dependency updates
  - Scans `apps/` and `clusters/` directories for image/version updates

## Key Concepts

### Flux CD Integration
- **GitRepository**: Each cluster has a Git source pointing to this repo (see `clusters/*/git-repository.yaml`)
- **Kustomization overlays**: Cluster-specific patches and customizations applied via Kustomize
- **Update frequency**: 1 minute reconciliation interval (see git-repository.yaml)

### Kustomization Files
Files follow standard Kustomize structure with:
- `resources:` - Lists of Kubernetes manifests to include
- `commonLabels:` - Labels applied to all resources (app.kubernetes.io/name, app.kubernetes.io/part-of)
- `patchesStrategicMerge:` and `patchesJson6902:` for cluster-specific overrides
- `configMapGenerator:` and `secretGenerator:` for ConfigMaps and Secrets

### Secret Management
- Secrets stored in `clusters/*/overlays/apps/*-secret.yaml` are encrypted with SOPS
- Each cluster has its own age key in `.sops.yaml`
- Only the `data` and `stringData` sections are encrypted (metadata remains readable)
- **Never commit unencrypted secrets** - use `sops` to edit encrypted files

### Apps vs Clusters Pattern
- **Base apps** (`apps/`) contain generic, cluster-agnostic definitions
- **Cluster overlays** (`clusters/*/overlays/apps/`) apply cluster-specific customization
- This separation allows the same app to run on multiple clusters with different configs

## Common Tasks

### Adding a New Application
1. Create `apps/my-app/` directory
2. Add `kustomization.yaml` referencing resource files or Helm release
3. Create `clusters/{ocr1,rp}/overlays/apps/my-app/` with cluster-specific kustomization
4. If app needs secrets, create `clusters/{ocr1,rp}/overlays/apps/my-app/secrets/*-secret.yaml`
   - Encrypt with: `sops clusters/ocr1/overlays/apps/my-app/secrets/my-secret.yaml`
If the app has no official helm chart it is recommended to use bwj-s app-template to create an helm chart for the app. Take a look at the [values.yaml](https://raw.githubusercontent.com/bjw-s-labs/helm-charts/refs/heads/main/charts/library/common/values.yaml) for the underlying fields. Take a look at `apps/vikunja` for an example.

### Updating Image Versions
- Use semantic version tags in manifests; Renovate will auto-detect and create PRs
- Check `renovate.json` for which file patterns are scanned

### Editing Encrypted Secrets
```bash
# Edit encrypted secret (opens in editor, re-encrypts on save)
sops clusters/ocr1/overlays/apps/my-app/secrets/my-secret.yaml

# View unencrypted content
sops -d clusters/ocr1/overlays/apps/my-app/secrets/my-secret.yaml
```

### Validating Manifests
Most editors support Kubernetes YAML validation. Flux will validate on push, but catch errors locally first:
- Ensure `apiVersion`, `kind`, `metadata.name` are present in all manifests
- Verify `kustomization.yaml` `resources:` paths are correct
- Check that all referenced ConfigMaps/Secrets exist

## Important Conventions

- **YAML files**: Use `.yaml` extension (some apps use `.yml`, but `.yaml` is preferred)
- **Naming**: Follow Kubernetes naming conventions (lowercase, hyphens, no underscores)
- **Labels**: All resources must include `app.kubernetes.io/name` and `app.kubernetes.io/part-of` labels
- **Namespace isolation**: Each app gets its own namespace (created in overlay)
- **No inline comments in Kustomization**: Keep `kustomization.yaml` readable; complex logic goes in separate manifests

## Tools & Dependencies

- **kubectl** - For manual validation (not required for this repo, but useful: `kubectl apply --dry-run=client -f file.yaml`)
- **kustomize** - Used by Flux to generate final manifests (verify: `kustomize build clusters/ocr1/overlays/apps/my-app`)
- **sops** - Required to edit encrypted secrets (install: https://github.com/mozilla/sops)
- **age** - Encryption backend for sops (installed with sops)
- **Flux CLI** - Optional for local validation (flux check will validate against Flux CRDs)

## File Format Consistency

When editing YAML files:
- Use 2-space indentation (consistent across repo)
- Keep resources in alphabetical order when listed
- Use `---` separator between multiple resources in same file
- Quote string values containing special chars or that look like numbers/booleans

## Secret Encryption Keys

The `.sops.yaml` configuration defines:
- `laptop` - Local development machine
- `ocr1`, `tiny1`, `tiny2` - OCR1 cluster machines
- `rp` - RP cluster machine

When creating new secrets for a cluster, ensure the key aliases are properly referenced in creation rules. The SOPS encryption is not automatic on commit so ensure everything is encrypted.
