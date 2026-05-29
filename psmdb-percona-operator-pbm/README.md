# PSMDB pbm Blueprint (Pattern 5 — MinIO Keeper)

Kasten blueprint for **sharded** Percona Server for MongoDB (PSMDB) clusters that use
**Percona Backup for MongoDB (pbm) to a local MinIO instance**. Kasten snapshots the MinIO
PVC (which contains the full pbm archive), while pbm handles consistent cross-shard backup
and restore.

## Why this blueprint instead of [psmdb-percona-operator](../psmdb-percona-operator/)

The [psmdb-percona-operator](../psmdb-percona-operator/) blueprint (Pattern 1 — quiesce a replica)
targets **unsharded** replica sets. Extending it to sharded clusters is prohibitively complex:
you would need to quiesce every shard's secondary simultaneously, track multiple PVC names, and
exclude the non-quiesced PVCs from each shard at restore time.

This blueprint delegates that entire complexity to the PSMDB operator. pbm performs a
cluster-wide consistent snapshot that covers all shards and the config server replica set
atomically. The blueprint is a thin orchestrator — it creates a `PerconaServerMongoDBBackup` CR,
waits for it to complete, and emits the backup name as an artifact. Restore is equally simple:
create a `PerconaServerMongoDBRestore` CR and wait.

## Versions

| Component | Version |
|---|---|
| Kubernetes | `1.32` (EKS) |
| Kasten | `8.5.4` |
| PSMDB Operator / Helm chart | `1.22.0` / `psmdb-db 1.22.2` |
| Percona Server for MongoDB | `8.0.19-7` |
| Percona Backup for MongoDB | `2.12.0` |
| MinIO | `RELEASE.2025-04-22T22-12-26Z` |

## Pattern

**Pattern 5 — pbm backup via a local MinIO keeper (vendor operator data mover).**

A dedicated MinIO `Deployment` (the *keeper*) with a permanent PVC stores the pbm archive.
The `backupPrehook` blueprint action:
1. Creates a `PerconaServerMongoDBBackup` CR to trigger a pbm logical backup across all shards.
2. Polls until the backup `status.state` reaches `ready`.
3. Emits the backup CR name, pbm internal name, cluster name, and namespace as restore-point
   artifacts.

Kasten then snapshots the MinIO PVC (and only that PVC — PSMDB data PVCs are excluded via
policy filter). The MinIO snapshot contains a self-consistent pbm archive: a full logical backup
of all shards and the config server.

On restore, the `restorePosthook` action:
1. Waits for the PSMDB cluster to reach `ready` state after Kasten restores the CR and MinIO PVC.
2. Creates a `PerconaServerMongoDBRestore` CR referencing the artifact backup name.
3. Polls until the restore `status.state` reaches `ready`.

**Why this pattern beats direct PVC snapshotting for sharded clusters:**
pbm provides a cluster-wide consistent point-in-time snapshot that spans all replica sets.
Snapshotting the raw MongoDB data PVCs directly across shards would capture each shard at a
slightly different point in time, producing an inconsistent cluster state on restore (missing
cross-shard transactions, inconsistent chunk metadata in the config server).

## Architecture

```
┌───────────────────────────────────────────┐
│  Namespace: psmdb-test                    │
│                                           │
│  ┌──────────┐   ┌─────────┐   ┌─────────┐ │
│  │ cfg rs   │   │  rs0    │   │ mongos  │ │
│  │ (3 pods) │   │ (3 pods)│   │ (1 pod) │ │
│  └──────────┘   └─────────┘   └─────────┘ │
│       │               │                   │
│  PVCs (excluded   pbm backup-agent        │
│  from snapshot)   on every pod            │
│                        │                  │
│                        ▼                  │
│                  ┌──────────┐             │
│                  │  MinIO   │             │
│                  │  Keeper  │             │
│                  │   PVC    │◄────────────┼── Kasten snapshots this
│                  └──────────┘             │
└───────────────────────────────────────────┘
```

The BlueprintBinding targets Deployments labelled `psmdb-minio: "true"`. The MinIO Deployment
name must follow the convention `<cluster-name>-minio` — the blueprint derives the PSMDB cluster
name by stripping the `-minio` suffix.

## Blueprint actions

| Action | Hook | What it does |
|---|---|---|
| `backupPrehook` | Before PVC snapshot | Creates `PerconaServerMongoDBBackup` CR, waits for pbm to complete, emits `backupCRName`, `pbmName`, `clusterName`, `namespace` as restore-point artifacts |
| `restorePosthook` | After PVC restore | Waits for PSMDB cluster ready, creates `PerconaServerMongoDBRestore` CR, polls until `ready`, waits for cluster to return to `ready` |
| `delete` | On restore-point deletion | Runs `pbm delete-backup` to remove archive data from MinIO, then deletes the `PerconaServerMongoDBBackup` CR; silently no-ops if the namespace or pod no longer exists |

> **Why a `delete` action is necessary**: deleting a Kasten restore point only removes the MinIO
> PVC snapshot. The pbm archive data inside MinIO and the `PerconaServerMongoDBBackup` CR are not
> cleaned up automatically. Without the `delete` action, the MinIO bucket grows unboundedly and
> old CRs accumulate. The `delete` action runs `pbm delete-backup` directly in the backup-agent
> container (which has pbm pre-configured) and removes the Kubernetes CR.

## Prerequisites

- PSMDB operator installed (`percona/psmdb-operator` Helm chart, `watchAllNamespaces=true`).
- `michaelcourcy/kasten-tools:8.5.2` image available (adds `kubectl` to the base image).
- Blueprint and BlueprintBinding deployed in the `kasten-io` namespace.
- Kasten backup policy configured with the PVC filter described below.

## Deploy the operator

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

helm upgrade --install psmdb-operator percona/psmdb-operator \
  --namespace psmdb-operator --create-namespace \
  --version 1.22.0 \
  --set watchAllNamespaces=true \
  --wait
```

## Deploy the workload

### 1. Create the namespace

```bash
kubectl create namespace psmdb-test
```

### 2. Deploy the MinIO keeper

Review [minio-keeper.yaml](minio-keeper.yaml) and replace `storageClassName: ebs-sc` with a
CSI storage class that supports snapshots on your cluster.

> **Storage class**: `ebs-sc` is the AWS EBS CSI storage class used in our test environment.
> Replace it with a storage class that supports CSI snapshots on your cluster
> (e.g. `managed-csi` on AKS, `standard-rwo` on GKE, your custom class on bare-metal, etc.).
> The class must have a matching `VolumeSnapshotClass` registered with Kasten.
> Do **not** use legacy in-tree classes (e.g. `gp2` on AWS) — they do not support CSI snapshots.

```bash
kubectl apply -f minio-keeper.yaml
kubectl wait deployment my-psmdb-psmdb-db-minio -n psmdb-test --for=condition=Available --timeout=3m
```

### 3. Create the MinIO bucket

```bash
kubectl run mc-init -n psmdb-test --image=minio/mc:latest --restart=Never --command -- \
  /bin/sh -c "mc alias set local http://my-psmdb-psmdb-db-minio:9000 minioadmin minioadmin123 \
    && mc mb local/pbm-backup && mc ls local"
sleep 10
kubectl logs mc-init -n psmdb-test
kubectl delete pod mc-init -n psmdb-test
```

### 4. Deploy the sharded PSMDB cluster

Review [psmdb-values.yaml](psmdb-values.yaml) and replace `storageClassName: ebs-sc` with your
CSI storage class (same requirement as above).

```bash
helm upgrade --install my-psmdb percona/psmdb-db \
  --namespace psmdb-test \
  --version 1.22.2 \
  -f psmdb-values.yaml \
  --wait --timeout=15m
```

Wait for the cluster to become ready:

```bash
kubectl wait psmdb my-psmdb-psmdb-db -n psmdb-test \
  --for=jsonpath='{.status.state}'=ready --timeout=15m
```

The cluster topology once ready:

| Component | Pods | Role |
|---|---|---|
| Config server rs | `my-psmdb-psmdb-db-cfg-0/1/2` | Config replica set |
| Shard rs0 | `my-psmdb-psmdb-db-rs0-0/1/2` | Data shard |
| Mongos | `my-psmdb-psmdb-db-mongos-0` | Query router |
| MinIO keeper | `my-psmdb-psmdb-db-minio-*` | pbm archive storage |

### 5. Create test data

Connect via the mongos router:

```bash
DBADMIN_USER=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_USER}" | base64 -d)
DBADMIN_PASS=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_PASSWORD}" | base64 -d)

kubectl exec -n psmdb-test my-psmdb-psmdb-db-mongos-0 -c mongos -- \
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
kubectl exec -n psmdb-test my-psmdb-psmdb-db-mongos-0 -c mongos -- \
  mongosh "mongodb://localhost:27017/admin" -u "$DBADMIN_USER" -p "$DBADMIN_PASS" \
  --quiet --eval '
var docs = db.getSiblingDB("testdb").customers.find().toArray();
print("count:", docs.length);
docs.forEach(d => print(d.name, d.email, d.plan));
'
```

## Deploy the blueprint

```bash
kubectl apply -f blueprint.yaml
kubectl apply -f blueprintbinding.yaml
```

## Kasten policy configuration

Create a backup policy targeting the `psmdb-test` namespace with a **Location** profile
(required for Kanister phases). 

**Critical: exclude PSMDB data PVCs from the snapshot.** Only the MinIO PVC must be snapshotted.
All MongoDB data PVCs carry the label `app.kubernetes.io/name=percona-server-mongodb`. Configure
the policy's volume filter to exclude them:

```yaml
backupParameters:
  filters:
    excludeResources:
      - matchLabels:
          app.kubernetes.io/name: percona-server-mongodb
```

This filter serves two purposes:

- **Correctness at restore time** — if MongoDB data PVCs are restored alongside the MinIO PVC,
  the cluster would start with stale on-disk data that contradicts the pbm archive. The restore CR
  would then overwrite inconsistent data with the correct archive, but the operator behaviour
  during this transition is unpredictable.
- **Avoiding redundant storage** — all MongoDB data is fully represented in the pbm archive inside
  MinIO. Snapshotting the raw data PVCs adds cost with no recovery benefit: the MinIO PVC alone
  is sufficient for a complete restore.

## Verifying a backup

After a RunAction completes:

```bash
# Confirm the backup CR was created and completed
kubectl get psmdb-backup -n psmdb-test

# Check pbm archive in MinIO
MINIO_POD=$(kubectl get pods -n psmdb-test -l app=my-psmdb-psmdb-db-minio \
  -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n psmdb-test "$MINIO_POD" -- sh -c "
  mc alias set local http://localhost:9000 minioadmin minioadmin123 2>/dev/null
  mc find local/pbm-backup/ --name '*.pbm.json'
"

# Check pbm backup list via the backup-agent
kubectl exec -n psmdb-test my-psmdb-psmdb-db-rs0-0 -c backup-agent -- pbm list
```

## Restore

Restore is fully automated via the `restorePosthook`. Kasten restores the MinIO PVC (containing
the pbm archive), the PSMDB CR, Secrets, and other resources. Once the cluster reaches `ready`
state, the `restorePosthook` creates a `PerconaServerMongoDBRestore` CR and waits for it to
complete.

### Simulate data loss

Drop the test collection so the restore has something meaningful to bring back:

```bash
DBADMIN_USER=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_USER}" | base64 -d)
DBADMIN_PASS=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_PASSWORD}" | base64 -d)

kubectl exec -n psmdb-test my-psmdb-psmdb-db-mongos-0 -c mongos -- \
  mongosh "mongodb://localhost:27017/admin" -u "$DBADMIN_USER" -p "$DBADMIN_PASS" \
  --quiet --eval '
db.getSiblingDB("testdb").customers.drop();
print("count after drop:", db.getSiblingDB("testdb").customers.countDocuments());
'
```

### Triggering a RestoreAction

```bash
# List available restore points
kubectl get restorepoint -n psmdb-test

RESTORE_POINT=$(kubectl get restorepoint -n psmdb-test \
  -o jsonpath='{.items[-1].metadata.name}')

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
    name: ${RESTORE_POINT}
    namespace: psmdb-test
  targetNamespace: psmdb-test
  # No profile needed — Kasten extracts the location profile from the RestorePointContent.
EOF
```

### Verify restore

```bash
kubectl wait psmdb my-psmdb-psmdb-db -n psmdb-test \
  --for=jsonpath='{.status.state}'=ready --timeout=10m

DBADMIN_USER=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_USER}" | base64 -d)
DBADMIN_PASS=$(kubectl -n psmdb-test get secret my-psmdb-psmdb-db-secrets \
  -o jsonpath="{.data.MONGODB_DATABASE_ADMIN_PASSWORD}" | base64 -d)

kubectl exec -n psmdb-test my-psmdb-psmdb-db-mongos-0 -c mongos -- \
  mongosh "mongodb://localhost:27017/admin" -u "$DBADMIN_USER" -p "$DBADMIN_PASS" \
  --quiet --eval '
var docs = db.getSiblingDB("testdb").customers.find().toArray();
print("count:", docs.length);
docs.forEach(d => print(d.name, d.email, d.plan));
'
```

## pbm backup lifecycle and storage growth

Each Kasten backup creates one full pbm logical backup in MinIO. pbm does not yet support
incremental logical backups (physical incremental backups require WiredTiger checkpoints and
are available separately, but not via the operator CR mechanism).

**Storage growth**: each backup adds a full copy of all collection data. For test datasets this
is under 100 KiB; for production datasets size it accordingly.

**Garbage collection**: when a Kasten restore point is deleted, the blueprint `delete` action
runs `pbm delete-backup` inside the `backup-agent` container to remove the corresponding archive
from MinIO and deletes the `PerconaServerMongoDBBackup` CR. This keeps the live MinIO bucket
bounded to the number of active Kasten restore points.

## Tearing down

```bash
kubectl delete psmdb my-psmdb-psmdb-db -n psmdb-test 2>/dev/null || true
kubectl wait pods -n psmdb-test -l app.kubernetes.io/instance=my-psmdb-psmdb-db \
  --for=delete --timeout=3m 2>/dev/null || true
kubectl delete namespace psmdb-test
kubectl delete blueprint psmdb-pbm-blueprint -n kasten-io
kubectl delete blueprintbinding psmdb-pbm-blueprint-binding -n kasten-io
kubectl delete restorepointcontent -l k10.kasten.io/appNamespace=psmdb-test
```
