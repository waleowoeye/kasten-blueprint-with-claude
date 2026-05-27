# Couchbase Autonomous Operator — Kasten Blueprint

## Pattern

**Pattern 4 — Database dump or snapshot on a permanent Keeper PVC (PVC mounted by keeper)**

A dedicated `cb-example-keeper` Deployment mounts the `cb-example-keeper` PVC permanently.
The `backupPrehook` uses `KubeExec` to run `cbbackupmgr backup` inside the keeper pod,
which connects to the Couchbase cluster over the network and writes an incremental backup to
`/backup/data/kasten-repo`. Kasten then snapshots both the data PVC and the backup PVC.

On restore, Kasten restores the backup PVC from its snapshot. The `restorePosthook` then:
1. Resumes the CouchbaseCluster (if it was paused by `restorePrehook`)
2. Waits for the cluster to become `Available`
3. Runs `cbbackupmgr restore` from the keeper pod to replay all backup data into Couchbase

**Why this pattern?**
- Couchbase pods are managed directly by the operator (no StatefulSet to modify).
  A keeper Deployment is the natural isolation boundary.
- `cbbackupmgr` is available inside the `couchbase/server` image already used by the cluster —
  no custom image is needed for the keeper.
- The backup PVC pre-exists at Kasten discovery time, so it is included in the restore point
  without needing a BackupAction preHook.
- The keeper Deployment provides a stable BlueprintBinding target, satisfying the
  Single Responsibility Principle (SRP) and Open/Closed Principle (OCP).

**Naming convention and multi-keeper support**

The keeper Deployment and its PVC are named after the workload they serve, suffixed with
`-keeper` (`cb-example-keeper` for the `cb-example` cluster). When multiple Couchbase clusters
exist in the same namespace, deploy one keeper per cluster following the same convention.

The BlueprintBinding targets the label `couchbase-keeper: "true"` — not a specific deployment
name — so a single binding covers every keeper in the fleet. Each keeper's `env:` block carries
`USERNAME`, `PASSWORD`, and `CLUSTER` for its specific cluster. `KubeExec` phases inherit these
variables from the pod environment, so the blueprint needs no `objects:` block and no per-cluster
ConfigMap or Secret reference.

**Backup is incremental:** `cbbackupmgr` performs a full backup on the first run,
then incremental backups on subsequent runs. Each restore replays the full chain
from the archive, giving a consistent point-in-time view.

---

## Versions

| Component | Version |
|---|---|
| Kubernetes | 1.32 (EKS) |
| Kasten | 8.5.4 |
| Couchbase Operator Helm chart | `couchbase-operator-2.90.0` |
| Couchbase Operator | 2.9.0 |
| Couchbase Server | 7.6.6 |

---

## Prerequisites

### Install the Couchbase Autonomous Operator

```bash
# Add the Couchbase Helm repository
helm repo add couchbase https://couchbase-charts.storage.googleapis.com/
helm repo update

# Install the operator into the application namespace
helm install couchbase-operator couchbase/couchbase-operator \
  --namespace couchbase-test \
  --create-namespace
```

### Deploy the Couchbase Cluster

Create the admin secret and the CouchbaseCluster:

```bash
kubectl create secret generic cb-example-auth \
  --namespace couchbase-test \
  --from-literal=username=Administrator \
  --from-literal=password=password

kubectl apply -n couchbase-test -f - <<'EOF'
apiVersion: couchbase.com/v2
kind: CouchbaseCluster
metadata:
  name: cb-example
  namespace: couchbase-test
spec:
  image: couchbase/server:7.6.6
  security:
    adminSecret: cb-example-auth
  buckets:
    managed: true
  servers:
  - name: all-services
    size: 1
    services: [data, index, query]
    volumeMounts:
      default: data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      storageClassName: ebs-sc   # adapt to your cluster — see note below
      resources:
        requests:
          storage: 5Gi
  networking:
    exposeAdminConsole: false
EOF

# Wait for the cluster to become available
kubectl wait couchbasecluster cb-example -n couchbase-test \
  --for=jsonpath='.status.conditions[?(@.type=="Available")].status=True' \
  --timeout=300s
```

> **Storage class**: `ebs-sc` is the AWS EBS CSI storage class used in our test environment.
> Replace it with a CSI storage class that supports snapshots on your cluster
> (e.g. `managed-csi` on AKS, `standard-rwo` on GKE).
> The class must have a matching `VolumeSnapshotClass` registered with Kasten.
> Do **not** use legacy in-tree classes (e.g. `gp2` on AWS) — they do not support CSI snapshots.

### Create test data

```bash
# Create JSON documents in the default bucket
CB_POD=$(kubectl get pod -n couchbase-test -l couchbase_cluster=cb-example -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n couchbase-test $CB_POD -- \
  /opt/couchbase/bin/cbq -u Administrator -p password -e http://localhost:8093 \
  --script="UPSERT INTO \`default\` (KEY, VALUE)
    VALUES ('user:alice', {'name':'Alice','role':'admin'}),
    VALUES ('user:bob', {'name':'Bob','role':'dev'}),
    VALUES ('user:carol', {'name':'Carol','role':'ops'});"

# Verify
kubectl exec -n couchbase-test $CB_POD -- \
  curl -s -u Administrator:password \
  "http://localhost:8093/query/service" \
  -d 'statement=SELECT * FROM `default`' | python3 -m json.tool
```

### Deploy the backup keeper

```bash
# Create the backup PVC and the keeper Deployment (credentials are injected via env vars)
kubectl apply -f keeper.yaml
```

### Deploy the blueprint and BlueprintBinding

```bash
kubectl apply -f blueprint.yaml
kubectl apply -f blueprintbinding.yaml
```

---

## Blueprint actions

| Action | Phase | Function | Description |
|---|---|---|---|
| `backupPrehook` | `cbBackup` | `KubeExec` | Creates backup repo if absent; runs `cbbackupmgr backup` (full on first run, incremental on subsequent runs) inside the keeper pod. Uses `$USERNAME`, `$PASSWORD`, `$CLUSTER` env vars injected into the keeper container. Calls `sync` after backup completes to flush the OS page cache to the EBS volume — without this, cbbackupmgr's sqlite index files may be 0-byte on the snapshot, causing "invalid Rift index" errors on restore. |
| `backupPosthook` | `cbVerify` | `KubeExec` | Runs `cbbackupmgr info` to confirm the backup was written to the PVC before Kasten takes the PVC snapshot. |
| `restorePrehook` | `quiesceNamespace` | `KubeTask` | Scales all Deployments and StatefulSets in the namespace to 0 replicas and force-kills any remaining pods. This stops the Couchbase operator from recreating workloads or PVCs while Kasten deletes and restores them. **Not triggered in Kasten ≤ 8.5.x — see warning below.** |
| `restorePosthook` | `resumeCluster` | `KubeTask` | Derives the cluster name from the keeper Deployment name (strips `-keeper` suffix), patches `spec.paused=false`, then polls until the cluster is `Available`. |
| `restorePosthook` | `cbRestore` | `KubeExec` | Polls the Couchbase REST API until it accepts connections (cluster `Available` status does not guarantee the REST API is ready). Clears stale archive locks, then runs `cbbackupmgr restore --force-updates --restore-partial-backups --purge` from the keeper pod. `--purge` clears any incomplete restore-progress state left in the backup PVC by a previous failed attempt. Uses `$USERNAME`, `$PASSWORD`, `$CLUSTER` env vars from the keeper container. |

---

## ⚠️ `restorePrehook` not triggered in Kasten ≤ 8.5.x

**`restorePrehook` is a known no-op in Kasten ≤ 8.5.x.** The blueprint implements it for
forward compatibility, but it will not execute until a future Kasten release.

Until the fix ships, **you must pause the CouchbaseCluster manually** before triggering a
Kasten restore, otherwise the Couchbase operator will race against Kasten's PVC deletion and
the restore will time out.

### Manual pre-restore steps (required until Kasten fix ships)

The goal is to scale everything down so Kasten can freely delete and restore PVCs without
operator reconciliation conflicts. Scaling down the operator Deployments prevents them from
recreating workload pods or PVCs while Kasten operates.

```bash
NAMESPACE=couchbase-test

# 1. Scale down all Deployments and StatefulSets in the namespace
#    (this includes the operator, admission controller, and keeper)
kubectl scale deployment --all -n $NAMESPACE --replicas=0
kubectl scale statefulset --all -n $NAMESPACE --replicas=0

# 2. Wait for pods to terminate, then force-kill any stragglers
kubectl wait pod --all -n $NAMESPACE --for=delete --timeout=120s 2>/dev/null || true
kubectl delete pod --all -n $NAMESPACE --force --grace-period=0 2>/dev/null || true

# 3. Trigger the Kasten restore (replace placeholders)
kubectl get restorepoint -n $NAMESPACE        # pick the restore point to use
kubectl create -f - --validate=false <<EOF
apiVersion: actions.kio.kasten.io/v1alpha1
kind: RestoreAction
metadata:
  generateName: couchbase-restore-
  namespace: $NAMESPACE
spec:
  subject:
    apiVersion: apps.kio.kasten.io/v1alpha1
    kind: RestorePoint
    name: <RESTORE_POINT_NAME>
    namespace: $NAMESPACE
  targetNamespace: $NAMESPACE
  profile:
    name: <LOCATION_PROFILE_NAME>   # type=Location profile, e.g. "us-east-1"
    namespace: kasten-io
EOF

# 4. After the restore completes, the restorePosthook will automatically:
#    - Scale everything back up (Deployments and StatefulSets are restored to their
#      original replica counts by the spec restore)
#    - Wait for the CouchbaseCluster to be Available
#    - Clear any stale cbbackupmgr archive lock from the restored PVC
#    - Run cbbackupmgr restore --force-updates --restore-partial-backups to reload the data
```

---

## Cleanup

### Remove the test namespace and workloads

```bash
# Remove the operator Helm release (and the CouchbaseCluster, keeper, etc.)
helm uninstall couchbase-operator -n couchbase-test

# Delete the namespace (also removes the PVCs)
kubectl delete namespace couchbase-test
```

### Remove blueprint resources from kasten-io

```bash
kubectl delete blueprint couchbase-blueprint -n kasten-io
kubectl delete blueprintbinding couchbase-blueprintbinding -n kasten-io
```

### Remove Kasten restore points for this namespace

```bash
kubectl delete restorepointcontent \
  -l k10.kasten.io/appNamespace=couchbase-test
```
