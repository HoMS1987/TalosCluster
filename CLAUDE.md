# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A GitOps-driven Kubernetes cluster running on **Talos Linux**, managed by **Flux CD**. All cluster state is declared in this repo; Flux reconciles from Git every 10 minutes. There are no build steps or test suites — changes take effect when pushed to `main` and Flux picks them up.

## Essential Tools

- `flux` — reconcile cluster state, check Kustomization/HelmRelease status
- `kubectl` — inspect live cluster resources
- `talosctl` — manage Talos OS nodes (upgrades, reboots, config changes)
- `sops` — decrypt/encrypt secrets (requires `age.agekey` locally, gitignored)
- `talhelper` — generate Talos machine configs from `talconfig.yaml`

Key commands from `Notes.md`:
```bash
# Force Flux to re-sync immediately
flux reconcile source git cluster -n flux-system

# Talos node management
talosctl -n <node-ip> reboot
talosctl -n <node-ip> upgrade --image <image>
```

## Repository Layout

```
clusters/main/
├── talos/
│   ├── talconfig.yaml          # Talos node definitions (versions, IPs, extensions)
│   └── patches/                # OS-level patches (gc, nvidia)
└── kubernetes/
    ├── flux-entry.yaml         # Root Flux Kustomization — entry point for all k8s resources
    ├── apps/                   # User-facing applications (Nextcloud, Plex, Immich, etc.)
    ├── core/                   # Cluster-wide services (Blocky DNS, MetalLB config, ClusterIssuers)
    ├── system/                 # Infrastructure operators (cert-manager, CloudNativePG, Prometheus, Longhorn)
    ├── networking/             # Ingress controllers (nginx-external, nginx-internal)
    └── kube-system/            # System components (Cilium, metrics-server, node-feature-discovery)
repositories/
├── helm/                       # Helm repository sources
├── git/                        # Git repository sources (this repo, TrueCharts)
└── oci/                        # OCI sources (Flux manifests)
.github/
├── workflows/                  # placeholder.yaml (stub CI) + automerge.yaml
└── renovate.json5              # Renovate config (extends TrueCharts + flux + kubectl rules)
```

## Standard App Structure

Every application follows this pattern:

```
apps/{app-name}/
├── kustomization.yaml          # Points to ./app
├── ks.yaml                     # Flux Kustomization resource
└── app/
    ├── kustomization.yaml      # Lists all resources
    ├── namespace.yaml
    └── helm-release.yaml       # HelmRelease with values
```

The `ks.yaml` references a `sourceRef` to this git repo and a `path` to `./app`. It enables `prune: true` so deleted resources are cleaned up. SOPS decryption is configured at the root level and inherited.

## Secrets Management (SOPS + Age)

Sensitive values are encrypted in-place using SOPS. The `.sops.yaml` defines which YAML keys to encrypt by regex: `password`, `token`, `secret`, `cert`, `key`, `data`, `stringData`, etc.

- **Cluster-wide variables**: `clusters/main/clusterenv.yaml` — fully encrypted ConfigMap with VIPs, domains, credentials, S3 tokens, etc. Substituted into manifests via Flux's `postBuild.substituteFrom`.
- **App secrets**: typically in `app/*.secret.yaml` files.
- To edit an encrypted file: `sops <file>` (opens in `$EDITOR`, re-encrypts on save).
- Never commit decrypted secret content. `age.agekey` is in `.gitignore`.

## Flux Dependency and Substitution Pattern

Flux Kustomizations use `dependsOn` to enforce ordering (e.g., `cert-manager` before apps that need certificates). Variable substitution comes from two ConfigMaps:
- `cluster-config` (from `clusterenv.yaml`) — IPs, domains, credentials
- `upgrade-settings` (from `upgradesettings.yaml`) — Talos/Kubelet version pins

Variables are referenced in manifests as `${VARIABLE_NAME}`.

## Renovate Automation

Renovate opens PRs automatically for:
- Helm chart version bumps (via `# renovate: registryUrl=...` comments in HelmRelease files)
- Container image tags (via `# renovate: depName=...` comments in talconfig and upgrade settings)
- Flux component updates

The GitHub Actions `automerge.yaml` workflow auto-merges Renovate PRs using squash-merge after the placeholder CI passes. Minor updates auto-merge; patch/major updates for system components require manual approval.

## Talos Version Tracking

Talos OS and Kubernetes versions are pinned in two places:
1. `clusters/main/talos/talconfig.yaml` — `talosVersion` and `kubernetesVersion` with Renovate comments
2. `clusters/main/kubernetes/kube-system/talos-system-upgrade/upgradesettings.yaml` — runtime version substitution for system-upgrade-controller

## Storage Architecture

- **Longhorn**: Distributed block storage for stateful apps (replicated across nodes)
- **OpenEBS**: Local storage (hostpath) for apps that don't need replication
- **CloudNativePG**: PostgreSQL operator — apps requiring a database get a `Cluster` resource
- **Volsync**: Backup/restore of PVCs to Cloudflare S3 (R2); backup schedules defined per-app in `values.yaml`
