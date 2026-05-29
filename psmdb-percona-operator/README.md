# Kasten Blueprint — Percona Server for MongoDB (PSMDB) Operator

Backup and restore for MongoDB clusters managed by the
[Percona Operator for MongoDB](https://docs.percona.com/percona-operator-for-mongodb/).

## When this blueprint is needed — and when it is not

> **TL;DR — do not deploy this blueprint if your storage driver guarantees crash-consistent snapshots.**

Modern CSI drivers (AWS EBS, GCE PD, Azure Disk, Ceph RBD, and most enterprise SANs) capture
a point-in-time block image of the volume. MongoDB's WiredTiger storage engine uses a
write-ahead journal (WAL) stored on the same volume. On restart after a crash-consistent
snapshot, WiredTiger replays the journal and brings the storage engine to a fully consistent
state — exactly as if the server had been powered off suddenly. This is the standard cloud
backup method for MongoDB and is explicitly documented:
https://www.mongodb.com/docs/manual/tutorial/backup-with-filesystem-snapshots/

**If your storage is crash-consistent, `fsyncLock()` and this blueprint add nothing.**
Do not deploy the blueprint or the BlueprintBinding. Let Kasten snapshot the three PVCs
directly — WiredTiger handles recovery automatically.

Two prerequisites make this safe for the PSMDB operator:

1. **Journaling is always enabled.** WiredTiger journaling is on by default in MongoDB 4.0+ and
   cannot be disabled on replica-set members (`--nojournal` is rejected at startup). Verified:
   `getCmdLineOpts` shows no `--nojournal` flag; `journal/WiredTigerLog.*` exists on every node.

2. **Journal and data files are on the same volume.** The PSMDB operator configures a single PVC
   mounted at `dbPath: /data/db`. The `journal/` subdirectory lives inside that same mount point
   (`/dev/nvme4n1 → /data/db`). There is no `--journalPath` override that would split them.
   Verified by inspecting the running pod: `df -h /data/db/journal` reports the same device as
   `/data/db`. Both conditions satisfy the MongoDB snapshot requirement documented at
   https://www.mongodb.com/docs/manual/core/backups/#back-up-with-filesystem-snapshots.

**This blueprint is necessary when your storage backend does NOT guarantee crash-consistent
snapshots.** Some older NFS-backed, CIFS, or certain SAN/NAS implementations capture blocks
from different points in time within the same volume ("fuzzy" snapshot). WiredTiger cannot
reliably replay a journal whose blocks span multiple points in time, so MongoDB may fail to
start from a fuzzy snapshot.

`fsyncLock()` on a secondary flushes all pending writes to disk and halts new I/O before
Kasten takes the snapshot. The result is a perfectly clean volume image that does not require
any journal replay — safe on any storage backend.

---

## Backup/Restore pattern

**Pattern 1 — Fence and quiesce a secondary replica.**

`backupPrehook` dynamically identifies a secondary member each time, runs `db.fsyncLock()`
on it, and records its PVC name as a restore-point artifact. Kasten snapshots **all three
PVCs** (the quiesced secondary provides a guaranteed clean anchor; the other two are also
captured). `backupPosthook` unlocks the secondary.

On restore, the operator must **manually** prepare before triggering the RestoreAction:

1. Delete the PSMDB cluster CR (stops all pods; the operator stops reconciling).
2. Delete all PVCs for the cluster.
3. Trigger a RestoreAction that **excludes the non-quiesced PVCs by name** — so only the
   quiesced PVC is restored from its snapshot.
4. The PSMDB operator (restored alongside the CR) creates fresh empty PVCs for the excluded
   members; they perform MongoDB initial sync from the quiesced node, which becomes primary.

Zero primary impact during backup: writes continue normally while the secondary is frozen.

### Why three PVCs are snapshotted but only one is used for restore

The quiesced secondary's PVC is the sole authoritative restore source. The other two snapshots
are taken alongside it because Kasten snapshots all PVCs it discovers at backup time — they
are present in the restore point but excluded from the RestoreAction. This is intentional:
excluding non-quiesced PVCs avoids any risk of fuzzy-snapshot data contaminating the cluster
on non-crash-consistent storage.

### Why the PVC label is not the tracking mechanism

`backupPrehook` labels the quiesced PVC with `kasten.io/psmdb-quiesced=true` for visibility,
but this label is set **after** Kasten's PVC discovery pass. Kasten captures PVC specs at
discovery time (before `backupPrehook` runs), so the label is not recorded in the restore
point and will not appear on the restored PVC. The **Kanister artifact**
`psmdbMeta.quiescedPVC` is the authoritative identifier — it is stored in the restore point
and is used to determine which PVCs to exclude from the RestoreAction.

---

## Blueprint actions

| Action | Function | What it does |
|---|---|---|
| `backupPrehook` | KubeTask | Removes stale `kasten.io/psmdb-quiesced` labels, picks a secondary (highest ordinal first), calls `db.fsyncLock()`, labels the pod (`kasten.io/psmdb-fsync-locked`) and the PVC (`kasten.io/psmdb-quiesced`), emits quiesced PVC name as restore-point artifact |
| `backupPosthook` | KubeTask | Finds the labeled pod, calls `db.fsyncUnlock()`, removes the pod label |

---

## Versions

| Component | Version |
|---|---|
| Kubernetes | `1.32.9` (EKS) |
| Kasten | `8.5.4` |
| PSMDB Operator / Helm chart | `1.22.0` |
| Percona Server for MongoDB | `8.0.19-7` |

---

## Prerequisites

### Operator namespace separation

Deploy the PSMDB operator in a dedicated namespace and workload clusters in separate
namespaces. This keeps operator lifecycle (GitOps / ArgoCD) independent of application
data managed by Kasten.

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

helm upgrade --install psmdb-operator percona/psmdb-operator \
  --namespace psmdb-operator --create-namespace \
  --version 1.22.0 \
  --set watchAllNamespaces=true \
  --wait
```

### Custom tool image

The blueprint uses `michaelcourcy/kasten-tools:8.5.2`, which adds `kubectl` to
`gcr.io/kasten-images/kanister-tools:8.5.2` (the base image ships without `kubectl`).
KubeTask phases that call `kubectl` must run in the `kasten-io` namespace for RBAC.

---

## Deploy the workload

```bash
kubectl create namespace psmdb-test

helm upgrade --install my-psmdb percona/psmdb-db \
  --namespace psmdb-test \
  --version 1.22.2 \
  --set replsets.rs0.size=3 \
  --set replsets.rs0.resources.requests.cpu=300m \
  --set replsets.rs0.resources.requests.memory=512Mi \
  --set replsets.rs0.resources.limits.cpu=500m \
  --set replsets.rs0.resources.limits.memory=1Gi \
  --set replsets.rs0.volumeSpec.pvc.storageClassName=ebs-sc \
  --set replsets.rs0.volumeSpec.pvc.resources.requests.storage=5Gi \
  --set pmm.enabled=false \
  --set backup.enabled=false \
  --set sharding.enabled=false \
  --wait --timeout=10m
```

> **Storage class**: `ebs-sc` is the AWS EBS CSI storage class used in our test environment.
> Replace it with a storage class that supports CSI snapshots on your cluster
> (e.g. `managed-csi` on AKS, `standard-rwo` on GKE, your custom class on bare-metal, etc.).
> The class must have a matching `VolumeSnapshotClass` registered with Kasten.
> Do **not** use legacy in-tree classes (e.g. `gp2` on AWS) — they do not support CSI snapshots.

Wait for the cluster to become ready:

```bash
kubectl get psmdb -n psmdb-test -w
# STATUS column should reach: ready
```

### Create test data

```bash
DBADMIN_USER=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_USER}" | base64 -d)
DBADMIN_PASS=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_PASSWORD}" | base64 -d)

kubectl exec -n psmdb-test my-psmdb-psmdb-db-rs0-0 -c mongod -- \
  mongosh "mongodb://localhost:27017/admin" -u "$DBADMIN_USER" -p "$DBADMIN_PASS" \
  --quiet --eval '
db.getSiblingDB("testdb").customers.insertMany([
  {name:"Alice",email:"alice@example.com",plan:"gold"},
  {name:"Bob",email:"bob@example.com",plan:"silver"},
  {name:"Carol",email:"carol@example.com",plan:"gold"}
]);
print("count:", db.getSiblingDB("testdb").customers.countDocuments());
'
```

### Verify test data

```bash
kubectl exec -n psmdb-test my-psmdb-psmdb-db-rs0-0 -c mongod -- \
  mongosh "mongodb://localhost:27017/admin" -u "$DBADMIN_USER" -p "$DBADMIN_PASS" \
  --quiet --eval '
var docs = db.getSiblingDB("testdb").customers.find().toArray();
print("count:", docs.length);
docs.forEach(d => print(d.name, d.email, d.plan));
'
```

### Delete the workload

```bash
helm uninstall my-psmdb -n psmdb-test
kubectl delete namespace psmdb-test
kubectl delete restorepointcontent -l k10.kasten.io/appNamespace=psmdb-test
```

---

## Deploy the blueprint

```bash
kubectl apply -f blueprint.yaml
kubectl apply -f blueprintbinding.yaml
```

---

## Backup policy

Create a backup policy targeting the `psmdb-test` namespace with a **Location** profile
(required for Kanister phases). No PVC filtering configuration is needed — all three PVCs
are snapshotted, and the restore procedure excludes the non-quiesced ones by name.

### Finding the quiesced PVC name

After a backup completes, find the quiesced PVC name from the executor logs:

```bash
kubectl logs -n kasten-io -l component=executor --tail=10000 | \
  grep -o '"quiescedPVC":"[^"]*"'
```

The artifact is also visible in the Kasten UI under the restore point's artifact details.

### Verify the backup hooks ran

```bash
kubectl logs -n kasten-io -l component=executor --tail=10000 | \
  grep -E "quiesceSecondary|unquiesceSecondary|fsyncLock|fsyncUnlock|Quiesced PVC"
```

---

## Restore

Restore requires manual preparation before triggering the RestoreAction. The goal is to
ensure that only the quiesced PVC's data drives recovery.

### Step 1 — Find the quiesced PVC name

From the executor logs or the Kasten UI artifact, note the quiesced PVC name, e.g.:
`mongod-data-my-psmdb-psmdb-db-rs0-2`

The non-quiesced PVCs are all other members, e.g.:
`mongod-data-my-psmdb-psmdb-db-rs0-0` and `mongod-data-my-psmdb-psmdb-db-rs0-1`

### Step 2 — Delete the PSMDB CR and all PVCs

Delete the PSMDB cluster CR **first** so the operator stops reconciling and cannot
recreate PVCs while you delete them.

```bash
APP_NS=psmdb-test
CR_NAME=my-psmdb-psmdb-db

kubectl delete psmdb ${CR_NAME} -n ${APP_NS}

# Wait for all pods to stop
kubectl wait --for=delete pod -l app.kubernetes.io/instance=${CR_NAME} \
  -n ${APP_NS} --timeout=120s

# Delete all PVCs
kubectl delete pvc -n ${APP_NS} -l app.kubernetes.io/instance=${CR_NAME}
```

### Step 3 — Trigger the RestoreAction

Exclude the non-quiesced PVCs by name so only the quiesced PVC is restored from its
snapshot. The PSMDB operator (restored with the CR) creates fresh empty PVCs for the
excluded members; they sync from the quiesced node via MongoDB initial sync.

```bash
# List available restore points
kubectl get restorepoint -n psmdb-test

kubectl create -f - --validate=false <<EOF
apiVersion: actions.kio.kasten.io/v1alpha1
kind: RestoreAction
metadata:
  generateName: restore-
  namespace: psmdb-test
spec:
  subject:
    apiVersion: apps.kio.kasten.io/v1alpha1
    kind: RestorePoint
    name: <RESTORE_POINT_NAME>
    namespace: psmdb-test
  targetNamespace: psmdb-test
  filters:
    excludeResources:
    - name: "mongod-data-my-psmdb-psmdb-db-rs0-0"   # adjust to non-quiesced PVC names
    - name: "mongod-data-my-psmdb-psmdb-db-rs0-1"   # adjust to non-quiesced PVC names
  # No profile needed — Kasten extracts the location profile from the RestorePointContent.
EOF
```

The RestoreAction will restore exactly 1 volume (the quiesced PVC). The PSMDB operator
reconciles the restored CR, creates fresh empty PVCs for the two excluded members, and
MongoDB initial sync populates them from the quiesced node (which becomes primary).

### Verify restore

```bash
DBADMIN_USER=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_USER}" | base64 -d)
DBADMIN_PASS=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_PASSWORD}" | base64 -d)

kubectl exec -n psmdb-test my-psmdb-psmdb-db-rs0-0 -c mongod -- \
  mongosh "mongodb://localhost:27017/admin" -u "$DBADMIN_USER" -p "$DBADMIN_PASS" \
  --quiet --eval '
var docs = db.getSiblingDB("testdb").customers.find().toArray();
print("count:", docs.length);
docs.forEach(d => print(d.name, d.email, d.plan));
'
```
