# Longhorn — distro-specific addon-owned resources

The `addons-longhorn` appset syncs `distros/<distro>/` (the cluster's `distro` label) with the longhorn
addon's own Application, so these resources exist **iff Longhorn is enabled** on a cluster of that distro.

Use it for what "Longhorn on distro X" needs regardless of which cluster it is — e.g. unflagging the
distro's own default StorageClass, since Longhorn's chart makes `longhorn` the default and two defaults
make Kubernetes pick the newest, unpredictably.

**Every distro label in use needs a folder** (`resources: []` is fine), otherwise the longhorn
Application fails with "app path does not exist". Current labels: `microk8s` (on-prem), `kind` (KinD).
