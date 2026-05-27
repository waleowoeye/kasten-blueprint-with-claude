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

Then create the Kasten backup policy. The policy must:

1. Use a **Location profile** (not Infra) as `backupParameters.profile` so that
   Kanister phases have a profile available at restore time.
2. **Exclude every ECK-managed resource from the backup** — the MinIO
   keeper PVC is the only thing we want in the restore point. ECK rebuilds
   the StatefulSet/Services/Secrets/Pods from the live Elasticsearch CR;
   re-applying them from a restore point causes a destructive rolling
   reconciliation that disrupts the cluster and breaks master-quorum
   (see "Why the broad exclusion" below).
3. **Exclude the Elasticsearch CR itself** — same reason; it stays in
   the live cluster.

Example policy filter:

```yaml
backupParameters:
  filters:
    excludeResources:
      # Skip every ECK-managed resource (StatefulSet, Services, Secrets,
      # PVCs, ConfigMaps, Pods) — they all carry this label.
      - matchLabels:
          common.k8s.elastic.co/type: elasticsearch
      # The Elasticsearch CR doesn't carry the label above, so exclude
      # by group+resource.
      - group: elasticsearch.k8s.elastic.co
        resource: elasticsearches
```

### Why the broad exclusion

If the backup captures the StatefulSet and ECK objects and Kasten re-applies
them on restore, ECK reconciles the cluster: it scales the StatefulSet down
to 1 (because of its "one master at a time" safety), waits for pod-0 to
become Ready, and then refuses to scale up because pod-0 cannot form a
quorum without pod-1 and pod-2. The live cluster never recovers without
manual `kubectl scale sts ... --replicas=3` intervention.

Excluding all ECK-managed resources means **the restore touches only the
MinIO keeper PVC** (and the keeper Deployment, which is idempotent against
its own spec). The live ES cluster is never disrupted; the blueprint just
orchestrates the ES snapshot/restore API against it.

## Manual workaround for restore (Kasten ≤ 8.5.x)

`restorePrehook` is defined in the blueprint for forward compatibility, but
the Kasten executor does **not** yet trigger it in 8.5.x — Kasten
engineering confirmed it ships in a near-future release. The blueprint works
around this by repeating the same close-indices step inside
`restorePosthook`, so **no manual operator action is required for restore**
on current Kasten.

When `restorePrehook` is eventually enabled, the `restorePosthook` fallback
close becomes a no-op (already-closed indices stay closed; the
`expand_wildcards=open` query parameter limits the call to open indices).

## End-to-end test (Step 5)

Run a full backup/restore cycle through Kasten:

```bash
# 1. Trigger a backup via Kasten policy (UI or RunAction)
#    Wait for status: Complete

# 2. Drop test-index to simulate data loss
ELASTIC_PASS=$(kubectl get secret es-cluster-es-elastic-user \
  -n elasticsearch-eck-minio -o jsonpath='{.data.elastic}' | base64 -d)
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" -X DELETE "https://localhost:9200/test-index"

# 3. Trigger a Kasten RestoreAction (UI or YAML — see CLAUDE.md for the spec).
#    The live ES cluster is NOT disrupted; only the MinIO PVC is restored
#    from the Kasten snapshot.

# 4. After restore completes, verify documents are back.
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" -X POST "https://localhost:9200/test-index/_refresh"
kubectl exec -n elasticsearch-eck-minio sts/es-cluster-es-default -- \
  curl -sk -u "elastic:${ELASTIC_PASS}" "https://localhost:9200/test-index/_count"
# Expected: {"count":5,...}

# 5. Retire the restore point to trigger the blueprint's `delete` action.
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

All three lifecycle phases pass end-to-end:

- **Backup** — `backupPrehook` creates a timestamped ES snapshot
  (`kasten-YYYYMMDDHHMMSS`), syncs the keeper filesystem, emits the
  snapshot name as a Kanister artifact. Kasten then snapshots the MinIO
  PVC. The ES data PVCs and StatefulSet are never touched.
- **Restore** — Kasten restores **only** the MinIO PVC; the live ES
  cluster keeps running. `restorePosthook` discovers the snapshot's user
  indices, deletes any pre-existing copies in the live cluster, and
  restores by exact name list via the ES REST API. Verified: a 5-doc
  `test-index` is recovered identically with `include_global_state:false`,
  and ES pods retain their uptime through the entire restore.
- **Delete** — When the RestorePointContent is removed, Kasten invokes
  the blueprint's `delete` action, which calls
  `DELETE /_snapshot/kasten-repo/<snapshot>` against the live ES.
  Verified end-to-end against a real Kasten policy.
