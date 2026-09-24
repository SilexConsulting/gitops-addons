# GitOps addons catalogue

This repository is an addons catalogue intended to be
used in conjunction with https://github.com/SilexConsulting/gitops-control-plane

Please consult the README.md in that repository for more information.

## Resources (GIT-20)

Raw manifests (not Helm charts) are declared under `resources/` trees and reconciled last, gated
per-cluster by the `enable_resources` label. This repo hosts **two** categories, in two sibling
trees so they never collide:

| Tree (per scope) | Category | ApplicationSet → Application |
|---|---|---|
| `environments/<scope>/resources/` | **cluster-wide** (no owner) — e.g. an `IngressClass`, `StorageClass` | root-level `resources` (in `gitops-control-plane`) → `cluster-<name>-resources` |
| `environments/<scope>/addons-resources/` | **addon-owned** — e.g. a CloudNativePG `Cluster` | `addons-resources` (`bootstrap/resources.yaml`) → `cluster-<name>-addons-resources` |

`<scope>` is `default` (all clusters), `<env>`, then `clusters/<cluster>` (most specific wins).
Each dir has a root `kustomization.yaml` **listing** the manifests to apply — a manifest that
isn't listed (e.g. an addon resource where the operator is disabled) is simply never rendered.

Cluster-wide resources live here (the addons/platform repo) rather than in the workloads repo.
Full design: `gitops-control-plane/docs/git-20-resources-design.md`.
