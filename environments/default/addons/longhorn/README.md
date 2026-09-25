# Longhorn add-on

Replicated block storage (`driver.longhorn.io`) with snapshots and off-cluster backups to S3.
Enabled per cluster with the `enable_longhorn` label (appset: `gitops/addons/oss/longhorn/addons-longhorn-appset.yaml`).

| Where | What |
|---|---|
| `values.yaml` (this folder) | Catalogue defaults: GitOps-safe settings only |
| `resources/` | Commented-out reference setup: backup/snapshot RecurringJobs, a `Retain` StorageClass, ESO credentials |
| `distros/<distro>/` | Resources every cluster of that distro needs while Longhorn is installed (see `distros/README.md`) |
| `<private>/addons/clusters/<cluster>/longhorn/` | Cluster values (data path, replica count, backup target, UI exposure) + addon-owned resources + `secrets.enc.yaml` |

## Safety — read before changing or removing anything

- **The chart owns the CRDs, and the CRDs hold every volume.** The appset sets `preserveResourcesOnDeletion: true`,
  so deleting the ApplicationSet/Application never cascades.
- **Never `helm uninstall`** a release this add-on has adopted. To drop Helm's record of an adopted release, delete its
  `sh.helm.release.v1.longhorn.*` Secrets instead.
- The chart's `longhorn-uninstall` Job is a Helm `pre-delete` hook, which Argo CD 3.x runs as a **PreDelete** hook when the
  Application is deleted. `defaultSettings.deletingConfirmationFlag: false` (pinned in `values.yaml`) makes that Job
  refuse, so deleting the Application cannot wipe data. Keep it false.
- `preUpgradeChecker.jobEnabled: false`: Longhorn's guidance for GitOps installs.
- **Replica count ≤ number of storage nodes** (`persistence.defaultClassReplicaCount`, `defaultSettings.defaultReplicaCount`,
  and every extra StorageClass's `numberOfReplicas`). Otherwise volumes stay `degraded` forever. Existing volumes keep
  their count; lower it with `kubectl -n longhorn-system patch volumes.longhorn.io <vol> --type merge -p '{"spec":{"numberOfReplicas":2}}'`.
- From chart **1.12.1**, `networkPolicies.restrictInternalTraffic` defaults to `true` and renders NetworkPolicies even with
  `networkPolicies.enabled: false`. Check volume I/O and RWX mounts after upgrading into it.

## Adopting an existing (hand-installed) Longhorn

1. Render the chart **at the live version** with the live user values (`helm get values longhorn -n longhorn-system`)
   plus this folder's `values.yaml`, drop hooks, and `kubectl diff --server-side` against the cluster. Only
   `deleting-confirmation-flag` should appear, in the `longhorn-default-setting` ConfigMap.
2. Pin the cluster to that version from the private repo (a merge-generator patch on `addonChartVersion`), and put the
   live values in the cluster's private `values.yaml`.
3. Enable `enable_longhorn`. The first sync is a no-op takeover; the PostSync `longhorn-post-upgrade` Job runs once.
4. Delete the `sh.helm.release.v1.longhorn.*` Secrets, then upgrade deliberately by dropping the pin.

## Backups to S3

1. **Bucket + credentials.** Use a dedicated bucket and an IAM user allowed only `s3:ListBucket` / `s3:GetBucketLocation` on
   the bucket and `s3:GetObject` / `PutObject` / `DeleteObject` on `bucket/*`. Put the keys in a Secret in `longhorn-system`
   with keys `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` (plus `AWS_ENDPOINTS` for non-AWS S3). Never commit it in
   clear: SOPS-encrypt it in the private repo and apply it out-of-band (`make private-secrets CLUSTER=<name> CONTEXT=<ctx>`
   in gitops-control-plane), or use external-secrets.
   **Create the Secret before** the target points at it.
2. **Target**, in the cluster's values:
   ```yaml
   defaultBackupStore:
     backupTarget: s3://<bucket>@<region>/
     backupTargetCredentialSecret: <secret-name>
   ```
   Check: `kubectl -n longhorn-system get backuptargets.longhorn.io default -o jsonpath='{.status.available}'` → `true`.
3. **Schedules** are `RecurringJob`s (addon-owned resources, see `resources/*.example.yaml`). Volumes without explicit
   recurring-job labels are in the `default` group. A StorageClass can put its volumes in groups with
   `recurringJobSelector`.

### Manual backup

```yaml
apiVersion: longhorn.io/v1beta2
kind: Snapshot
metadata: {name: my-snap, namespace: longhorn-system}
spec: {volume: <longhorn-volume-name>, createSnapshot: true}
---
apiVersion: longhorn.io/v1beta2
kind: Backup            # apply once the Snapshot is readyToUse
metadata: {name: my-backup, namespace: longhorn-system, labels: {backup-volume: <longhorn-volume-name>}}
spec: {snapshotName: my-snap}
```

Wait for `kubectl -n longhorn-system get backups.longhorn.io my-backup -o jsonpath='{.status.state}'` → `Completed`;
`.status.url` is the backup's URL. (The UI API's `snapshotBackup` action needs an explicit snapshot name; the CRs are simpler.)
Detached volumes are attached for the snapshot automatically.

### Restore into a new volume

A restore needs the backup's **`BackupVolume`** record, which Longhorn creates only when it next polls the target
(`pollInterval`, 5 min by default). Until then the webhook rejects the restore with `cannot get backup volume … not found`.
Check with `kubectl -n longhorn-system get backupvolumes.longhorn.io`.

```yaml
apiVersion: longhorn.io/v1beta2
kind: Volume
metadata: {name: restored-vol, namespace: longhorn-system}
spec:
  fromBackup: "<backup .status.url>"
  size: "<bytes, as a string>"      # >= the original volume's size
  numberOfReplicas: 2
  dataLocality: disabled            # REQUIRED: if unset, the webhook assumes strict-local and allows only 1 replica
  frontend: blockdev
  dataEngine: v1
  accessMode: rwo
  backupTargetName: default
```

The restore is finished when the volume is `detached` with `restoreRequired=false`. Bind it with a static PV and PVC:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: {name: restored-vol}
spec:
  capacity: {storage: 1Gi}
  accessModes: [ReadWriteOnce]
  persistentVolumeReclaimPolicy: Retain
  storageClassName: longhorn-static
  csi: {driver: driver.longhorn.io, fsType: ext4, volumeHandle: restored-vol}
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: {name: restored, namespace: <ns>}
spec: {accessModes: [ReadWriteOnce], storageClassName: longhorn-static, volumeName: restored-vol, resources: {requests: {storage: 1Gi}}}
```

(The Longhorn UI can do the same: *Backup → Restore*, then *Create PV/PVC*.)

### DR drill (prove backups restore)

1. In a scratch namespace, write random data plus its checksum to a new PVC:
   `head -c 52428800 /dev/urandom > /data/blob && sync && sha256sum /data/blob > /data/blob.sha256`.
2. Take a manual backup (above) and wait for `Completed`.
3. Wait for the `BackupVolume` to appear, then restore into a **new** volume and bind it (above).
4. Mount it and compare `sha256sum /data/blob` with `/data/blob.sha256`: they must match.
5. Clean up: delete the scratch namespace, the static PV and restored Longhorn volume, and the test `BackupVolume`.
   Deleting it removes the test's objects from S3; confirm with `aws s3 ls s3://<bucket>/ --recursive --summarize`.

## Troubleshooting

- **Backups fail with `tls: handshake failure`, but the target shows `available`.** The upload is done by **one of the
  volume's replicas**, i.e. the instance-manager on *that replica's* node, not by longhorn-manager. If that pod's
  `/etc/resolv.conf` has an extra search domain that the node got from DHCP and that domain has a DNS **wildcard**, then
  `<bucket>.s3.<region>.amazonaws.com` (fewer dots than `ndots:5`) resolves to the wildcard host first. Check from the
  instance-manager: the relative name fails TLS, while the absolute one (trailing dot) works:
  ```sh
  kubectl -n longhorn-system exec <instance-manager> -- sh -c 'grep search /etc/resolv.conf;
    curl -so /dev/null -w "%{http_code} %{errormsg}\n" https://<bucket>.s3.<region>.amazonaws.com/;
    curl -so /dev/null -w "%{http_code} %{errormsg}\n" https://<bucket>.s3.<region>.amazonaws.com./'
  ```
  Fix the node (e.g. netplan `dhcp4-overrides: {use-domains: false}`), then **restart that node's pods**: pods keep the
  `resolv.conf` they started with. See *Restarting an instance-manager*.
- **Restarting an instance-manager kills every engine and replica on that node.** Workloads whose volume **engine** is
  there get I/O errors mid-flight. Do it properly:
  1. `kubectl cordon <node>`;
  2. move the engines off: delete the pods using volumes whose `status.currentNodeID` is that node, one at a time,
     waiting for each to be Ready elsewhere (StatefulSets and CNPG instances are recreated; move a CNPG *replica*, or
     switch over first);
  3. delete the node's `instance-manager`, `longhorn-manager` and `longhorn-csi-plugin` pods;
  4. **`kubectl uncordon <node>` before waiting for the new instance-manager.** Longhorn does not recreate an
     instance-manager on a cordoned node (`Schedulable=False, KubernetesNodeCordoned`), so volumes stay degraded until
     you uncordon;
  5. the node's replicas then rebuild from the other copies (heavy I/O, safe).
