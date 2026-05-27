# CNPG Barman Blueprint (Pattern 5 — MinIO Keeper)

Kasten blueprint for CloudNativePG clusters that use **barman-cloud WAL archiving to a local MinIO instance**. Kasten snapshots the MinIO PVC (which contains the full barman archive), enabling Point-In-Time Recovery (PITR) from any Kasten restore point.

> ⚠️ **Deprecation notice**: barman-cloud integration in CNPG is deprecated and will be **fully removed in CNPG 1.30.0**. Consider migrating to the [cnpg/](../cnpg/) Pattern 1 blueprint (fence and quiesce a replica) for new deployments. This blueprint targets CNPG ≤ 1.29.x.

## Versions

| Component | Version |
|---|---|
| Kubernetes | `1.32` (EKS) |
| Kasten | `8.5.4` |
| CNPG operator / Helm chart | `1.29.0` / `cloudnative-pg 0.28.0` |
| PostgreSQL | `18.3` |
| MinIO | `RELEASE.2025-04-22T22-12-26Z` |

## Pattern

**Pattern 5 — Database backup via a local MinIO keeper (vendor operator data mover).**

A dedicated MinIO `Deployment` (the *keeper*) with a permanent PVC stores the CNPG barman archive. The `backupPrehook` blueprint action:
1. Creates a CNPG `Backup` CR to trigger a barman base backup.
2. Polls until the base backup completes.
3. Forces a WAL segment switch (`pg_switch_wal()`) so the post-backup checkpoint WAL is flushed to MinIO before Kasten snapshots the PVC.

Kasten then snapshots the MinIO PVC (and only that PVC — CNPG data PVCs are excluded via policy filter). The MinIO snapshot contains a self-consistent barman archive: a base backup plus all WALs up to the switch.

> **Why the WAL switch is necessary here, but not in a typical barman setup**
>
> When barman archives to an external object store (S3, GCS, Azure Blob), the final WAL segment
> produced by `pg_backup_stop()` will eventually arrive in the bucket on its own — the archiver
> keeps running, and recovery fetches WAL on demand. There is no snapshot deadline, so the timing
> of that last segment does not matter in practice.
>
> Here the situation is different. The Backup CR reaches `completed` once the **base backup files**
> are fully uploaded to MinIO. But `pg_backup_stop()` also generates a WAL record at the *stop
> LSN* — the exact position PostgreSQL must replay to reach a consistent state from that base
> backup. That record lands in the *currently-active, partially-filled* WAL segment on the primary
> pod's `pg_wal` directory. Because PostgreSQL's WAL archiver only ships **complete** 16 MB
> segments, that partial segment stays local and unarchived — potentially indefinitely on an idle
> cluster.
>
> Kasten snapshots the MinIO PVC immediately after the blueprint phase returns. The snapshot is
> frozen at that instant: no more WAL files will ever be added to it. If the stop-LSN WAL segment
> has not been archived yet, the snapshot is missing the one WAL piece that makes the base backup
> usable for recovery.
>
> `pg_switch_wal()` closes the current segment and hands it to the WAL archiver right now. The
> 15-second wait gives the archiver time to upload it to MinIO. After that, the snapshot is taken
> against a MinIO PVC that contains both the base backup and the complementary WAL needed to reach
> a consistent recovery point — which is exactly what a barman-based PITR restore requires.

**Why this pattern:**  
The goal is to give Kasten full ownership of data protection while keeping barman's scope strictly local to the namespace. Barman writes to a MinIO instance that lives in the same namespace — it never leaves it. Kasten then takes over everything beyond that boundary:

- **Encryption at rest** — Kasten encrypts snapshot data at the storage layer; barman has no equivalent.
- **Secure, authenticated data movement** — Kasten handles egress to the backup target; the application team never needs credentials for the external location profile.
- **Immutability** — Kasten enforces immutability at the backup target; barman retention policies alone cannot guarantee this.
- **Broad backup target support** — Kasten can protect to NFS, Veeam Vault, VBR, and other targets that barman-cloud does not support.
- **Incrementality** — Kasten's PVC snapshot mechanism is incremental; a plain barman export to an external store is a full transfer each time.
- **Local restore cache** — because MinIO is in the same namespace, a restore from the most recent restore point does not require fetching data from the external backup location — it reads directly from the local PVC snapshot, which is fast.
- **Credential isolation** — the application team only needs to create MinIO credentials inside their namespace. They never need to know or share the Kasten location profile credentials, which simplifies both security and configuration significantly.
- **Repeatable, self-contained configuration** — every component (MinIO Deployment, PVC, CNPG Cluster, Secrets, blueprint) lives entirely within the application namespace. The same manifests can be deployed verbatim in a new namespace just by changing the namespace name — nothing else needs to be adapted. This makes the pattern an ideal candidate for productisation with Helm: a single chart parameterised only on the namespace (and optionally the storage class) can provision a fully protected CNPG cluster in any environment.

## Architecture

```
┌──────────────────────────────────┐
│  Namespace: cnpg-barman          │
│                                  │
│  ┌──────────┐   WAL stream       │
│  │  CNPG    │──────────────────► │
│  │  Cluster │   barman-cloud     │
│  │          │   base backups     │
│  └──────────┘        │           │
│       │              ▼           │
│  PVCs (excluded   ┌────────┐     │
│  from snapshot)   │ MinIO  │     │
│                   │ Keeper │     │
│                   │  PVC   │◄────┼── Kasten snapshots this
│                   └────────┘     │
└──────────────────────────────────┘
```

The BlueprintBinding targets Deployments labelled `cnpg-barman-minio: "true"`. The MinIO Deployment name must follow the convention `<cluster-name>-minio` — the blueprint derives the CNPG cluster name by stripping the `-minio` suffix.

The CNPG Cluster must carry the label `cnpg-backup-pattern: barman-minio`, which prevents the [cnpg-blueprint-binding](../cnpg/blueprintbinding.yaml) (Pattern 1) from also binding to it.

## Blueprint actions

| Action | Hook | What it does |
|---|---|---|
| `backupPrehook` | Before PVC snapshot | Creates CNPG `Backup` CR, waits for completion, forces WAL switch; outputs `backupName` and `namespace` as restore-point artifacts |
| `delete` | On restore-point deletion | Deletes the CNPG `Backup` CR from the application namespace so the CNPG operator removes the corresponding barman data from MinIO; silently no-ops if the namespace or Backup CR no longer exists |

No restore hooks are implemented — PITR recovery is a **manual procedure** using a CNPG recovery cluster manifest (see [PITR Recovery Procedure](#pitr-recovery-procedure) below).

> **Why a `delete` action is necessary**: deleting a Kasten restore point only removes the MinIO PVC snapshot. The CNPG `Backup` CR and the barman base-backup data it references inside MinIO are not cleaned up automatically. Without the `delete` action, the MinIO bucket grows unboundedly. The `delete` action triggers the CNPG operator's normal garbage-collection path by deleting the `Backup` CR.

## Prerequisites

- CNPG operator installed (`cloudnative-pg` Helm chart).
- `michaelcourcy/kasten-tools:8.5.2` image available (adds `kubectl` to `gcr.io/kasten-images/kanister-tools:8.5.2`).
- Blueprint and BlueprintBinding deployed in `kasten-io` namespace.
- Kasten backup policy configured with the PVC filter described below.
- The Pattern 1 `cnpg-blueprint-binding` must have the `NotIn: barman-minio` exclusion condition (already present in `cnpg/blueprintbinding.yaml`).

## Deploying the workload

### 1. Create the namespace

```bash
kubectl create namespace cnpg-barman
```

### 2. Deploy MinIO keeper

Review [minio-keeper.yaml](minio-keeper.yaml) before applying — in particular the `storageClassName` on the PVC, which must be replaced with a CSI storage class that supports snapshots on your cluster.

> **Storage class**: `ebs-sc` is the AWS EBS CSI storage class used in our test environment.
> Replace it with a storage class that supports CSI snapshots on your cluster
> (e.g. `managed-csi` on AKS, `standard-rwo` on GKE, your custom class on bare-metal, etc.).
> The class must have a matching `VolumeSnapshotClass` registered with Kasten.
> Do **not** use legacy in-tree classes (e.g. `gp2` on AWS) — they do not support CSI snapshots.

```bash
kubectl apply -f minio-keeper.yaml
kubectl wait deployment pg-cluster-minio -n cnpg-barman --for=condition=Available --timeout=3m
```

### 3. Create the MinIO bucket

```bash
kubectl run mc-init -n cnpg-barman --image=minio/mc:latest --restart=Never --rm -it --command -- \
  /bin/sh -c "
    mc alias set local http://pg-cluster-minio:9000 minioadmin minioadmin123
    mc mb local/cnpg-backup
    mc mb local/cnpg-backup-recovered
    mc ls local
  "
```

### 4. Deploy the CNPG cluster

Review [cnpg-cluster.yaml](cnpg-cluster.yaml) and replace `storageClassName: ebs-sc` with your cluster's CSI storage class (same requirement as above).

```bash
kubectl apply -f cnpg-cluster.yaml
kubectl wait cluster.postgresql.cnpg.io pg-cluster -n cnpg-barman --for=condition=Ready --timeout=5m
```

### 5. Create test data

```bash
PRIMARY=$(kubectl get pods -n cnpg-barman \
  -l "cnpg.io/cluster=pg-cluster,cnpg.io/instanceRole=primary" \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n cnpg-barman "$PRIMARY" -- psql -U postgres -c "
  CREATE DATABASE kasten_test;
"

kubectl exec -n cnpg-barman "$PRIMARY" -- psql -U postgres -d kasten_test -c "
  CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary INTEGER,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
  );
  INSERT INTO employees (name, department, salary) VALUES
    ('Alice Martin',  'Engineering', 95000),
    ('Bob Chen',      'Engineering', 88000),
    ('Carol Smith',   'Marketing',   72000),
    ('David Johnson', 'Sales',       68000),
    ('Eve Williams',  'Engineering', 102000);
  SELECT * FROM employees ORDER BY id;
"
```

### 6. Deploy the blueprint and BlueprintBinding

```bash
kubectl apply -f ../cnpg/blueprintbinding.yaml   # ensure NotIn: barman-minio exclusion is present
kubectl apply -f blueprint.yaml -n kasten-io
kubectl apply -f blueprintbinding.yaml -n kasten-io
```

## Kasten policy configuration

Create a backup policy on the `cnpg-barman` namespace with a **location profile** (Kanister actions need it for `KubeTask`).

**Critical: exclude CNPG data PVCs from the snapshot.** Only the MinIO PVC must be snapshotted. Configure the policy's volume filter to exclude PVCs with the CNPG data-role label:

```yaml
backupParameters:
  filters:
    excludeResources:
      - matchLabels:
          cnpg.io/pvcRole: PG_DATA
```

This filter serves two purposes:

- **Correctness at restore time** — if the CNPG data PVCs are restored alongside the MinIO PVC, the operator would find PVCs with stale PostgreSQL data that predate the WAL archive. Any attempt to use the barman archive for recovery would fail or produce inconsistent results.
- **Avoiding redundant storage** — the CNPG data PVCs (primary and replica) are already fully represented in the MinIO barman archive. Snapshotting them adds cost and snapshot quota consumption with no recovery benefit: the MinIO PVC alone is sufficient for a complete restore.

## Understanding PITR range and WAL retention

This section explains how far back you can recover and to what precision, so you can size the barman `retentionPolicy` correctly relative to your Kasten backup schedule.

### What a Kasten restore point contains

A Kasten restore point is a self-contained, immutable snapshot of the MinIO PVC taken at backup time. It is important to understand that this PVC does not only contain the base backup just created by `backupPrehook` — it contains the **entire barman archive** that existed on MinIO at that moment:

- **All base backups** not yet pruned by the barman `retentionPolicy`.
- **All WAL files** archived since the oldest retained base backup.

WAL archiving is continuous: barman-cloud uploads a WAL segment to MinIO every time PostgreSQL seals one, 24 hours a day, independently of whether a base backup is running. By the time Kasten takes its snapshot, the MinIO PVC already holds a dense, uninterrupted WAL stream going back to the oldest retained base backup.

### Concrete example

Suppose Kasten runs daily backups and `retentionPolicy: "7d"`. On **January 3rd at 10:00** the `backupPrehook` creates a base backup and Kasten snapshots MinIO. That snapshot contains:

- The **January 2nd 10:00 base backup** (and earlier ones, within the 7-day window)
- Every WAL file produced between **January 2nd 10:00 and January 3rd 10:00** — including all transactions from January 3rd at 05:00

To recover to **January 3rd at 05:00** from that restore point, CNPG uses the **January 2nd base backup** as the starting point and replays WALs forward up to 05:00. The January 3rd base backup cannot be used as the start because it is dated *after* the target time — WAL replay only moves forward.

The restore point from January 3rd at 10:00 is therefore sufficient to recover to any moment between **January 2nd 10:00** and **January 3rd 10:00**.

### How far back can you go?

The oldest recoverable moment from a given restore point is the start time of the oldest base backup included in that MinIO snapshot. With daily Kasten backups and `retentionPolicy: "7d"`, the January 3rd restore point covers the last 7 days. The oldest Kasten restore point you still hold extends that window further back — each restore point carries its own independent WAL archive frozen at snapshot time.

### Sizing `retentionPolicy` relative to the Kasten backup interval

The barman `retentionPolicy` controls which base backups (and their associated WALs) are kept on the **live MinIO PVC**. Once a base backup falls outside the retention window, barman prunes it and the WALs that predate it from the live PVC.

The risk is this: if `retentionPolicy` is shorter than the Kasten backup interval, the previous base backup may be pruned from the live MinIO *before* the next Kasten snapshot captures it. The resulting snapshot would contain only the freshly-created base backup and no prior history — drastically reducing the PITR window available from that restore point.

**Rule of thumb: set `retentionPolicy` to at least 2× the Kasten backup interval.**

| Kasten backup frequency | Minimum recommended `retentionPolicy` |
|---|---|
| Daily | `3d` (default `7d` in this blueprint gives comfortable headroom) |
| Every 12 hours | `2d` |
| Weekly | `14d` |

### Summary table

| Question | Answer |
|---|---|
| How far back can I recover? | To the start of the oldest base backup in the restored MinIO snapshot, governed by `retentionPolicy` and the Kasten restore point age |
| How precisely can I target a recovery time? | Any transaction timestamp after the oldest base backup in the snapshot and before the last WAL in the snapshot |
| What governs how many restore points I can choose from? | Kasten policy retention (number or age of restore points kept by Kasten) |
| What happens if `retentionPolicy` < Kasten backup interval? | The previous base backup may be absent from the next snapshot, reducing PITR range to the interval since the last base backup only |

## Verifying a backup

After a RunAction completes, check:

```bash
# Confirm the Backup CR was created and completed
kubectl get backups.postgresql.cnpg.io -n cnpg-barman

# Verify base backup and WALs are in MinIO
MINIO_POD=$(kubectl get pods -n cnpg-barman -l app=pg-cluster-minio -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n cnpg-barman "$MINIO_POD" -- sh -c "
  mc alias set local http://localhost:9000 minioadmin minioadmin123 2>/dev/null
  mc ls --recursive local/cnpg-backup/ | tail -20
"
```

Expected output: a `base/` directory with a timestamped backup and a `wals/` directory with compressed WAL files, the last of which should be roughly 1–16 MiB (post-switch WAL).

## PITR Recovery Procedure

> **No restore hooks** — recovery is manual. The procedure below replaces what `restorePrehook`/`restorePosthook` would otherwise automate.

These steps recover the `cnpg-barman` namespace to the state captured in a Kasten restore point. You can optionally specify a `targetTime` to stop WAL replay before the end of the archive (true PITR).

### Step 1 — Record the target time (optional, for PITR)

Before triggering restore, record the timestamp you want to recover to. The timestamp must fall within the archive — i.e., after the base backup and before the last WAL in the archive.

```bash
# List available base backups to understand the archive range
MINIO_POD=$(kubectl get pods -n cnpg-barman -l app=pg-cluster-minio -o jsonpath='{.items[0].metadata.name}')
kubectl exec -n cnpg-barman "$MINIO_POD" -- sh -c "
  mc alias set local http://localhost:9000 minioadmin minioadmin123 2>/dev/null
  mc ls local/cnpg-backup/pg-cluster/base/
"
```

### Step 2 — Delete the CNPG cluster

CNPG must not be running when Kasten restores the MinIO PVC. Deleting the Cluster CR also removes all CNPG-managed PVCs (this is expected — the data PVCs are not snapshotted anyway).

```bash
kubectl delete cluster.postgresql.cnpg.io pg-cluster -n cnpg-barman
# Wait for all pg-cluster-N pods to disappear
kubectl wait pods -n cnpg-barman -l cnpg.io/cluster=pg-cluster --for=delete --timeout=3m
```

### Step 3 — Trigger Kasten restore

Derive the restore point, policy, and location profile automatically, then trigger the restore. Two `excludeResources` entries are applied so that no manual cleanup is needed afterward:

- **Label filter** `cnpg.io/cluster: pg-cluster` — skips CNPG-managed services and endpoints (CNPG refuses to own objects it did not create; they will be recreated by the operator).
- **Type + name filter** `postgresql.cnpg.io/clusters: pg-cluster` — skips the original Cluster CR (it will be replaced by `pg-cluster-restored` in the next step).

```bash
RESTORE_POINT=$(kubectl get restorepoint -n cnpg-barman \
  -o jsonpath='{.items[-1].metadata.name}')

POLICY=$(kubectl get restorepoint -n cnpg-barman "$RESTORE_POINT" \
  -o jsonpath='{.metadata.labels.k10\.kasten\.io/policyName}')

PROFILE=$(kubectl get policy "$POLICY" -n kasten-io \
  -o jsonpath='{.spec.actions[?(@.action=="backup")].backupParameters.profile.name}')

kubectl create -f - <<EOF
apiVersion: actions.kio.kasten.io/v1alpha1
kind: RestoreAction
metadata:
  generateName: restore-
  namespace: cnpg-barman
spec:
  subject:
    apiVersion: apps.kio.kasten.io/v1alpha1
    kind: RestorePoint
    name: ${RESTORE_POINT}
    namespace: cnpg-barman
  targetNamespace: cnpg-barman
  profile:
    name: ${PROFILE}
    namespace: kasten-io
  filters:
    excludeResources:
      - matchLabels:
          cnpg.io/cluster: pg-cluster
      - group: postgresql.cnpg.io
        resource: clusters
        name: pg-cluster
EOF
```

Wait for the RestoreAction to complete (100%). Kasten restores the MinIO PVC, Deployment, Secrets, ConfigMaps, and other resources — but not the CNPG-managed services or the original Cluster CR.

### Step 4 — Create a CNPG recovery cluster

Create a new Cluster CR named `pg-cluster-restored` that bootstraps from the barman archive in MinIO. The `externalClusters[].name` **must exactly match the original cluster name** (`pg-cluster`) because CNPG uses it as the barman server path in S3 (`s3://cnpg-backup/pg-cluster/`).

The `backup.barmanObjectStore.destinationPath` must point to a **different bucket** (`cnpg-backup-recovered`) — CNPG refuses to start if the write destination already contains an archive.

```bash
kubectl apply -f - <<EOF
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster-restored
  namespace: cnpg-barman
  labels:
    cnpg-backup-pattern: barman-minio
spec:
  instances: 2
  storage:
    size: 1Gi
    storageClass: ebs-sc       # replace with your CSI storage class
  bootstrap:
    recovery:
      source: pg-cluster
      # Uncomment and adjust for PITR — must be within the archive range:
      # recoveryTarget:
      #   targetTime: "2026-04-21 06:45:00"
  externalClusters:
    - name: pg-cluster          # must match original cluster name (S3 path)
      barmanObjectStore:
        endpointURL: http://pg-cluster-minio.cnpg-barman.svc:9000
        destinationPath: s3://cnpg-backup/
        s3Credentials:
          accessKeyId:
            name: pg-cluster-minio-creds
            key: rootUser
          secretAccessKey:
            name: pg-cluster-minio-creds
            key: rootPassword
        wal:
          compression: gzip
  backup:
    barmanObjectStore:
      endpointURL: http://pg-cluster-minio.cnpg-barman.svc:9000
      destinationPath: s3://cnpg-backup-recovered/   # different bucket — required
      s3Credentials:
        accessKeyId:
          name: pg-cluster-minio-creds
          key: rootUser
        secretAccessKey:
          name: pg-cluster-minio-creds
          key: rootPassword
      wal:
        compression: gzip
        maxParallel: 2
    retentionPolicy: "7d"
EOF
```

### Step 5 — Wait and verify

```bash
kubectl wait cluster.postgresql.cnpg.io pg-cluster-restored -n cnpg-barman \
  --for=condition=Ready --timeout=10m

PRIMARY=$(kubectl get pods -n cnpg-barman \
  -l "cnpg.io/cluster=pg-cluster-restored,cnpg.io/instanceRole=primary" \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n cnpg-barman "$PRIMARY" -- \
  psql -U postgres -d kasten_test -c "SELECT * FROM employees ORDER BY id;"
```

### PITR notes

- `recoveryTarget.targetTime` must be an ISO-8601 timestamp in UTC (`"YYYY-MM-DD HH:MM:SS"`).
- The timestamp must be **after** the base backup start time and **before** the last committed transaction in the WAL archive. If you specify a time beyond the archive, PostgreSQL will exit with `FATAL: recovery ended before configured recovery target was reached`.
- To recover to the very end of the archive (latest available point), **omit** `recoveryTarget` entirely.
- After recovery completes and the cluster is `Ready`, the recovered cluster will start archiving new WALs to `cnpg-backup-recovered`. If you want to resume archiving to the original bucket, recreate the cluster with `destinationPath: s3://cnpg-backup/` after manually clearing the archive (not recommended — prefer a fresh backup policy run instead).

## Tearing down

```bash
kubectl delete cluster.postgresql.cnpg.io pg-cluster -n cnpg-barman 2>/dev/null || true
kubectl delete namespace cnpg-barman
kubectl delete blueprint cnpg-barman-blueprint -n kasten-io
kubectl delete blueprintbinding cnpg-barman-blueprint-binding -n kasten-io

# Clean up restore point contents created by Kasten
kubectl delete restorepointcontent -l k10.kasten.io/appNamespace=cnpg-barman
```
