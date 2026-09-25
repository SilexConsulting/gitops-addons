# GitOps addons catalogue

This repository is an addons catalogue intended to be
used in conjunction with https://github.com/SilexConsulting/gitops-control-plane

Please consult the README.md in that repository for more information.

## Addon-owned resources (exist iff the addon is enabled)

Resources that belong to one addon (its namespace, generic CRs such as a `ClusterSecretStore`) live
**with the addon**, in `environments/default/addons/<addon>/resources/kustomization.yaml`. The addon's
ApplicationSet points its first source (the catalogue repo, `ref: values`) at that folder:

```yaml
sources:
  - repoURL: '{{metadata.annotations.addons_repo_url}}'
    targetRevision: '{{metadata.annotations.addons_repo_revision}}'
    ref: values
    path: environments/default/addons/{{values.addonChart}}/resources
  - chart: ...   # the addon's Helm chart
```

so the resources are synced by the addon's own Application: they are created **if and only if** the
addon is enabled on the cluster (no separate appset, nothing created on clusters without the addon).
Examples: `velero` (namespace), `metallb` (privileged `metallb-system` namespace). Site-specific
resources (e.g. a cluster's MetalLB address pools / BGP peers) come from the cluster's private repo in
the same way. Mark anything whose loss would hurt with `argocd.argoproj.io/sync-options: Delete=false`.

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
