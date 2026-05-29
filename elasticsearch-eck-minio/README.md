# Elasticsearch (ECK) + MinIO Keeper — Kasten Blueprint

Pattern 5 (local MinIO keeper) for an ECK-managed Elasticsearch cluster.
Elasticsearch's native S3 snapshot repository writes to a local MinIO instance;
Kasten snapshots the MinIO PVC, capturing the full ES snapshot archive in each
restore point.

This blueprint is the **Pattern 5 alternative** to the existing
[`elasticsearch-eck/`](../elasticsearch-eck/) blueprint (Pattern 3 — permanent
workload PVC with shared filesystem repository). Use this one when:

- You don't want to mount an RWX repository PVC across all ES nodes.
- You want **per-restore-point ES snapshot lifecycle** (snapshots survive
  across backups, are deleted when the corresponding Kasten restore point
  is retired).

## Versions

| Component | Version |
|---|---|
| Kubernetes | `1.32` (EKS) |
| Kasten | `8.5.7` |
| ECK operator | `2.16.1` |
| Elasticsearch | `8.17.0` |
| MinIO | `RELEASE.2025-04-22T22-12-26Z` |

## Pattern

**Pattern 5 — Database dump or snapshot via a local MinIO keeper.**

```
┌──────────────────────────────────────────────────────────────────┐
│  Namespace: elasticsearch-eck-minio                              │
│                                                                  │
│  Elasticsearch (3 nodes)         MinIO keeper Deployment         │
│  ┌──────────────────────┐        ┌──────────────────────┐        │
│  │ es-cluster-es-...    │ ──S3── │ es-cluster-minio     │        │
│  │ (data PVCs)          │ snap   │  ├─ minio container  │        │
│  └──────────────────────┘        │  └─ curl sidecar     │        │
│                                  │     (KubeExec target)│        │
│                                  └─────────┬────────────┘        │
│                                            │                     │
│                                  ┌─────────▼────────────┐        │
│                                  │ es-cluster-minio PVC │ ◄──┐   │
│                                  │ /data/es-snapshots/  │    │   │
│                                  └──────────────────────┘    │   │
└──────────────────────────────────────────────────────────────┼───┘
                                                               │
                                          Kasten CSI snapshot ─┘
```

- A MinIO `Deployment` with a permanent PVC stores the Elasticsearch S3
  snapshot archive. Kasten snapshots **only this PVC** (the ES data PVCs are
  excluded — they are recreated from the ES snapshot on restore).
- The keeper pod has a `curl` sidecar container so that the blueprint can
  `KubeExec` into the keeper to talk to the Elasticsearch HTTP API. Using
  `KubeExec` into the keeper rather than a `KubeTask` in `kasten-io` lets the
  blueprint use `.Object.metadata.labels` / `.Object.metadata.namespace`
  directly — the keeper Deployment **is** the BlueprintBinding context.
- Workload identity (ES cluster name + kind) is carried as labels on the
  MinIO Deployment so that one generic blueprint can serve many keepers.

### Blueprint actions

| Action | What it does |
|---|---|
| `backupPrehook` | `KubeExec` into the keeper's `curl` sidecar → `PUT /_snapshot/kasten-repo/kasten-<timestamp>` on the Elasticsearch HTTP API to create a new native snapshot, polls until `state=SUCCESS`, then emits the snapshot name as a `kasten-snapshot` artifact. |
| `backupPosthook` | No-op (placeholder for symmetry). |
| `restorePrehook` | Defined for forward compatibility (Kasten ≤ 8.5.x does not yet trigger it). When triggered: close all open indices to allow restore. |
| `restorePosthook` | Waits for the ES cluster to be healthy after Kasten restores the MinIO PVC, re-registers the snapshot repository (idempotent), then `POST /_snapshot/kasten-repo/<snapshot>/_restore` with `indices: "*"` and `include_global_state: false`. Polls until the restore completes. |
| `delete` | Called by Kasten when a restore point is retired. Receives the `kasten-snapshot` artifact and issues `DELETE /_snapshot/kasten-repo/<snapshot>`. Idempotent — treats `snapshot_missing_exception` as success. |

## Snapshot lifecycle (no per-backup cleanup)

Unlike [`elasticsearch-eck/`](../elasticsearch-eck/), this blueprint **never
deletes Elasticsearch snapshots during a backup run**. Each backup creates a
new ES snapshot with a unique name; the snapshot's identity is stored as a
Kanister artifact alongside the Kasten restore point. When Kasten retires the
restore point, the `delete` action removes the corresponding ES snapshot from
the live MinIO bucket.

The total size of the live MinIO bucket grows in proportion to the number of
live Kasten restore points (not to the absolute backup count), since each
retired restore point causes its ES snapshot to be deleted.

## Prerequisites

- ECK operator installed in `elastic-system`.
- A CSI storage class that supports volume snapshots. Examples used here:
  - `ebs-sc` — AWS EBS CSI on EKS.

  > **Storage class**: `ebs-sc` is the AWS EBS CSI storage class used in our
  > test environment. Replace it with a storage class that supports CSI
  > snapshots on your cluster (e.g. `managed-csi` on AKS, `standard-rwo` on
  > GKE, your custom class on bare-metal). The class must have a matching
  > `VolumeSnapshotClass` registered with Kasten. Do **not** use legacy
  > in-tree classes (e.g. `gp2`) — they do not support CSI snapshots.

## Deployment

### 1. Create the namespace and the MinIO keeper

```bash
kubectl create ns elasticsearch-eck-minio
kubectl apply -f minio-keeper.yaml
kubectl wait --for=condition=Available --timeout=180s \
  deployment/es-cluster-minio -n elasticsearch-eck-minio
```

### 2. Create the MinIO bucket used by Elasticsearch

```bash
kubectl exec -n elasticsearch-eck-minio deploy/es-cluster-minio -c minio -- \
  mc alias set local http://localhost:9000 minioadmin minioadminpassword
kubectl exec -n elasticsearch-eck-minio deploy/es-cluster-minio -c minio -- \
  mc mb --ignore-existing local/es-snapshots
```

### 3. Create the Elasticsearch cluster

```bash
kubectl apply -f elasticsearch.yaml
kubectl wait --for=jsonpath='{.status.health}'=green --timeout=600s \
  elasticsearch/es-cluster -n elasticsearch-eck-minio
```

### 4. Register the S3 snapshot repository (one-time setup)

The repository is registered once at deployment time and **not** managed by the
blueprint. Re-running this command is safe (PUT is idempotent).

```bash
ELASTIC_PASS=$(kubectl get secret es-cluster-es-elastic-user \
  -n elasticsearch-eck-minio -o jsonpath='{.data.elastic}' | base64 -d)

kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" \
       -X PUT "https://localhost:9200/_snapshot/kasten-repo" \
       -H 'Content-Type: application/json' \
       -d '{
             "type": "s3",
             "settings": {
               "bucket": "es-snapshots",
               "endpoint": "es-cluster-minio.elasticsearch-eck-minio.svc:9000",
               "protocol": "http",
               "path_style_access": true
             }
           }'
echo
```

Expected response: `{"acknowledged":true}`.

### 5. Create test data (small dataset for fast dev cycle)

```bash
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" \
       -X PUT "https://localhost:9200/test-index" \
       -H 'Content-Type: application/json' \
       -d '{"settings":{"number_of_shards":1,"number_of_replicas":1}}'
echo

for i in 1 2 3 4 5; do
  kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
    curl -sk -u "elastic:${ELASTIC_PASS}" \
         -X POST "https://localhost:9200/test-index/_doc" \
         -H 'Content-Type: application/json' \
         -d "{\"id\":${i},\"msg\":\"hello number ${i}\"}"
  echo
done

kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" \
       -X POST "https://localhost:9200/test-index/_refresh"
echo
```

Verify the documents are searchable:

```bash
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" \
       "https://localhost:9200/test-index/_search?pretty"
```

## Blueprint deployment

Apply the Blueprint and BlueprintBinding to `kasten-io`:

```bash
kubectl apply -f blueprint.yaml
kubectl apply -f blueprintbinding.yaml
```

The BlueprintBinding matches any `Deployment` carrying the label
`minio-s3-keeper=true` that does **not** already have a
`kanister.kasten.io/blueprint` annotation — so any other keeper Deployment
intended for a non-Elasticsearch S3-protocol workload (e.g.
[`cnpg-barman/`](../cnpg-barman/) uses its own label `cnpg-barman-minio`)
is unaffected.

Then create the Kasten backup policy. The policy needs:

1. A **Location profile** (not Infra) for both `backup` and `export` actions —
   blueprint phases need it, and DR restores need the data in S3.
2. An **`export` action** with `exportData: enabled: true` so the restore
   point survives namespace deletion. Without export, the in-cluster CSI
   snapshots are deleted along with their VolumeSnapshot CRs when the
   namespace is gone (`deletionPolicy: Delete` is the default on most CSI
   classes), and DR restores fail with `InvalidSnapshot.NotFound`.
3. An `excludeResources` filter that excludes **every ECK-managed resource**
   (label `common.k8s.elastic.co/type=elasticsearch`) — but **keeps the
   Elasticsearch CR**. ECK regenerates StatefulSet, Services, Secrets,
   ConfigMaps, Pods, and data PVCs from the CR on restore. Restoring them
   from the backup interferes with ECK's reconciliation (see "Why exclude
   ECK-managed resources" below).

```yaml
spec:
  actions:
    - action: backup
      backupParameters:
        profile:
          name: <location-profile>
          namespace: kasten-io
        filters:
          excludeResources:
            # Exclude every ECK-managed resource. ECK rebuilds them from
            # the ES CR on restore. The ES CR itself does NOT carry this
            # label and stays in the backup.
            - matchLabels:
                common.k8s.elastic.co/type: elasticsearch
    - action: export
      exportParameters:
        profile:
          name: <location-profile>
          namespace: kasten-io
        frequency: '@onDemand'
        exportData:
          enabled: true
```

### Why exclude ECK-managed resources (but keep the ES CR)

ECK normally creates and reconciles the StatefulSet, ES Services, ES
Secrets, ES ConfigMaps, ES Pods, and data PVCs — all derived from the
Elasticsearch CR. If the backup captures those derived resources and the
restore re-applies them:

- The restored StatefulSet creates pods bound to **either** the restored
  ES data PVCs (in-place) **or** fresh empty ones (DR). In the in-place
  case the pods read "I was node X in cluster [X,Y,Z]" from the persisted
  state and refuse to elect a master without quorum; ECK then scales the
  STS down to 1 in a "limit master node creation" safety, and the cluster
  deadlocks.
- In the DR case the restored objects pre-empt ECK's reconciliation —
  Kasten waits for the restored pods to be Ready before restoring the ES
  CR, but the pods can never be Ready without the CR's bootstrap config.

Excluding the ECK-managed resources lets the ES CR be the **single source
of truth** on both paths: ECK sees the (re-)applied CR, builds a fresh
StatefulSet with the correct bootstrap config, creates fresh empty data
PVCs, and elects a master cleanly. `restorePosthook` then orchestrates the
ES snapshot restore via the REST API to re-hydrate the data.

## Two restore paths

The blueprint covers two scenarios with one policy:

### In-place restore (live cluster still up)

Use case: rollback after data loss inside an otherwise-healthy cluster
(dropped index, bad migration, etc.).

**On Kasten ≥ (post-8.5.x where `restorePrehook` fires):** trigger a
`RestoreAction`. The blueprint's `restorePrehook` deletes the live ES CR;
ECK garbage-collects the StatefulSet, pods, and data PVCs via its
finalizer. Kasten then restores the CR + MinIO PVC; ECK rebuilds the
cluster from scratch with fresh empty PVCs; `restorePosthook` re-hydrates
the data from the restored MinIO snapshot.

**On Kasten ≤ 8.5.x (current),** `restorePrehook` does not fire. The
operator must run the equivalent step manually:

```bash
kubectl delete elasticsearch es-cluster -n elasticsearch-eck-minio
# Wait for ECK to fully garbage-collect (STS gone, PVCs gone):
until [ -z "$(kubectl get sts -n elasticsearch-eck-minio \
  -l elasticsearch.k8s.elastic.co/cluster-name=es-cluster --no-headers 2>/dev/null)" ]; do sleep 5; done
# Then trigger the Kasten RestoreAction.
```

### Disaster-recovery / migration restore (namespace destroyed, fresh cluster)

Use case: full namespace loss, cross-cluster migration, restore into a new
cluster after a regional outage.

`restorePrehook` does not fire (no live MinIO keeper in the target
namespace — Kasten silently skips the prehook by design, see
[kasten-kanister.md](../kasten-kanister.md#reserved-action-names--restore)).
Just trigger the restore against the **exported** RestorePointContent:

```bash
# 1. Find the exported RPC (the one with `k10.kasten.io/exportProfile` label set).
EXPORTED_RPC=$(kubectl get restorepointcontent \
  -l k10.kasten.io/appNamespace=elasticsearch-eck-minio \
  -o jsonpath='{.items[?(@.metadata.labels.k10\.kasten\.io/exportProfile)].metadata.name}' \
  | tr ' ' '\n' | tail -1)
echo "Exported RPC: ${EXPORTED_RPC}"

# 2. Re-create the namespace (Kasten won't if it's missing).
kubectl create ns elasticsearch-eck-minio

# 3. Re-create a RestorePoint that references the exported RPC.
cat <<EOF | kubectl create -f - --validate=false
apiVersion: apps.kio.kasten.io/v1alpha1
kind: RestorePoint
metadata:
  name: dr-restore-target
  namespace: elasticsearch-eck-minio
  labels:
    k10.kasten.io/appName: elasticsearch-eck-minio
    k10.kasten.io/appNamespace: elasticsearch-eck-minio
spec:
  restorePointContentRef:
    name: ${EXPORTED_RPC}
EOF

# 4. Trigger the RestoreAction.
cat <<EOF | kubectl create -f - --validate=false
apiVersion: actions.kio.kasten.io/v1alpha1
kind: RestoreAction
metadata:
  generateName: dr-restore-
  namespace: elasticsearch-eck-minio
spec:
  subject:
    apiVersion: apps.kio.kasten.io/v1alpha1
    kind: RestorePoint
    name: dr-restore-target
    namespace: elasticsearch-eck-minio
  targetNamespace: elasticsearch-eck-minio
  # No profile needed — Kasten extracts the location profile from the RestorePointContent.
EOF
```

Kasten restores the ES CR + MinIO keeper. ECK rebuilds the ES cluster
from scratch with fresh empty PVCs. `restorePosthook` waits for ES to be
reachable, **(re-)registers the snapshot repository** (idempotent, needed
because ES cluster state is brand-new in DR), and restores the user
indices from the snapshot. Verified end-to-end: a 5-doc `test-index` was
recovered identically after `kubectl delete ns elasticsearch-eck-minio`.

## End-to-end test (in-place flow)

```bash
# 1. Trigger a backup via Kasten policy (UI or RunAction)
#    Wait for both backup AND export to reach Complete.

# 2. Drop test-index to simulate data loss
ELASTIC_PASS=$(kubectl get secret es-cluster-es-elastic-user \
  -n elasticsearch-eck-minio -o jsonpath='{.data.elastic}' | base64 -d)
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" -X DELETE "https://localhost:9200/test-index"

# 3. On Kasten ≤ 8.5.x: manually delete the live ES CR (see "Two restore
#    paths" above). On post-8.5.x: skip — restorePrehook handles it.

# 4. Trigger a Kasten RestoreAction.

# 5. After restore completes, verify documents are back.
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" -X POST "https://localhost:9200/test-index/_refresh"
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" "https://localhost:9200/test-index/_count"
# Expected: {"count":5,...}

# 6. Retire the restore point to trigger the blueprint's `delete` action.
#    NOTE: `kubectl delete restorepoint <name>` alone does NOT fire the
#    delete action — Kasten only invokes it when the underlying
#    RestorePointContent (RPC) is removed. Either delete the RPC directly
#    or use the Kasten dashboard / RetireAction:
RPC=$(kubectl get restorepointcontent \
  -l k10.kasten.io/appNamespace=elasticsearch-eck-minio \
  -o jsonpath='{.items[0].metadata.name}')
kubectl delete restorepointcontent "${RPC}"
# Watch the delete actionset run and confirm the ES snapshot was removed:
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" \
       "https://localhost:9200/_snapshot/kasten-repo/_all"
```

## Teardown

```bash
# Delete any Kasten restore points that point at this app first, then:
kubectl delete -f blueprintbinding.yaml --ignore-not-found
kubectl delete -f blueprint.yaml --ignore-not-found
kubectl delete -f elasticsearch.yaml --ignore-not-found
kubectl delete -f minio-keeper.yaml --ignore-not-found
kubectl delete ns elasticsearch-eck-minio --ignore-not-found

# Clean up retained Kasten snapshot contents (replace selector if needed)
kubectl delete restorepointcontent \
  -l k10.kasten.io/appNamespace=elasticsearch-eck-minio --ignore-not-found
```

## Status

All lifecycle phases pass end-to-end:

- **Backup** — `backupPrehook` creates a timestamped ES snapshot
  (`kasten-YYYYMMDDHHMMSS`), syncs the keeper filesystem, emits the
  snapshot name as a Kanister artifact. Kasten then snapshots the MinIO
  PVC and exports the data to S3. The ES data PVCs are never touched.
- **In-place restore** — operator (or `restorePrehook` post-8.5.x)
  deletes the live ES CR; ECK garbage-collects everything. Kasten
  restores the CR + MinIO PVC; ECK rebuilds with fresh empty PVCs;
  `restorePosthook` re-hydrates the user indices via the ES REST API.
- **DR restore** — `kubectl delete ns elasticsearch-eck-minio` + trigger
  restore from the **exported** RPC. `restorePrehook` is silently
  skipped (no live keeper). ECK builds the cluster from scratch from
  the restored CR; `restorePosthook` (re-)registers the snapshot repo
  on the brand-new cluster and restores the user indices. Verified:
  5-doc `test-index` recovered identically after full namespace delete.
- **Delete** — When the RestorePointContent is removed, Kasten invokes
  the blueprint's `delete` action, which calls
  `DELETE /_snapshot/kasten-repo/<snapshot>` against the live ES.
  Verified end-to-end against a real Kasten policy.
