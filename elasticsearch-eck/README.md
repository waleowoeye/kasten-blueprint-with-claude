# Elasticsearch (ECK) — Kasten Blueprint

## Tested versions

| Component | Version |
|---|---|
| Kubernetes | 1.32 (EKS) |
| Kasten | 8.5.2 |
| ECK operator (eck-operator) | 2.16.1 |
| Elasticsearch | 8.17.0 |

## Pattern: Database snapshot on a permanent workload PVC (Pattern 3)

**Why Pattern 3 (permanent workload PVC), not Pattern 7 (temporary PVC)?**

Pattern 7 (Elasticsearch native snapshot to a new temporary PVC per backup) would be preferred
because Elasticsearch explicitly discourages filesystem-level PVC snapshots of its data volumes:
<https://www.elastic.co/docs/deploy-manage/tools/snapshot-and-restore#other-backup-methods>

However, the Elasticsearch **filesystem repository** requires `path.repo` to be set in
`elasticsearch.yml` at **pod startup**. ECK configures this via the `config` section of the
Elasticsearch CR, which triggers a rolling restart when changed. A per-backup temporary PVC scheme
would require two rolling restarts per backup cycle (add PVC → snapshot → remove PVC), which is
unacceptably disruptive.

Pattern 3 (permanent workload PVC) is used instead: a **permanent snapshot repository PVC** is declared once
in the Elasticsearch pod template as a one-time workload configuration change. This enables
Elasticsearch native snapshots (consistent, append-only) while Kasten CSI-snapshots the repo PVC
alongside the data PVCs. Kasten is the data mover; the ES native snapshot in the repo PVC is the
primary restore artefact.

### Snapshot repository PVC — storage class requirements

The Elasticsearch filesystem repository requires **all master and data nodes to access the same
shared filesystem path simultaneously**. For a multi-node cluster this means the snapshot
repository PVC must use a **ReadWriteMany (RWX) storage class**.

The RWX storage class must also support **CSI snapshots** so that Kasten can snapshot the repo PVC
as part of the restore point. Verify that:
- Your cluster has a `ReadWriteMany`-capable CSI provisioner (e.g. NFS CSI, CephFS, etc.).
- A `VolumeSnapshotClass` exists for that provisioner.
- Kasten has a Retain-policy clone VolumeSnapshotClass registered for that provisioner.

The manifests in `operator/` use `nfs-csi-1` as an example RWX storage class. Replace it with
the appropriate class for your environment.

### Backup flow

1. `backupPrehook`: Registers the filesystem snapshot repository at `/mnt/snapshot-repo`
   (idempotent); deletes the previous `kasten-snapshot` if it exists; takes a new ES native
   snapshot and waits for completion.
2. Kasten CSI-snapshots **all** PVCs in the namespace: the ES data PVCs and the snapshot repo PVC.
3. `backupPosthook`: Logs the final snapshot status.

### Restore flow

4. `restorePrehook`: Pauses ECK reconciliation and scales down all Elasticsearch StatefulSets so
   Kasten can safely replace PVC contents.
5. Kasten restores all PVC contents from backup.
6. `restorePosthook`: Resumes ECK reconciliation; waits for Elasticsearch to be healthy; registers
   the snapshot repository on the restored repo PVC; closes existing indices; runs
   `_snapshot/_restore` from the ES native snapshot; waits for restore completion. Using the ES
   native restore (not just the CSI-restored data PVC) is preferred because ES guarantees
   snapshot-level consistency.

---

## Blueprint actions

| Action | What it does |
|---|---|
| `backupPrehook` | Registers the filesystem snapshot repository at `/mnt/snapshot-repo`; deletes the previous `kasten-snapshot` for idempotency; takes a new Elasticsearch native snapshot and waits for completion |
| `backupPosthook` | Logs the final snapshot status after Kasten has taken CSI snapshots of all PVCs |
| `restorePrehook` | Annotates the Elasticsearch CR with `common.k8s.elastic.co/pause: "true"` to halt ECK reconciliation; scales down all Elasticsearch StatefulSets so Kasten can safely replace PVC contents |
| `restorePosthook` | Removes the pause annotation to resume ECK reconciliation; waits for the cluster to reach green or yellow health; registers the snapshot repository on the restored repo PVC; closes existing indices; runs ES native restore from `kasten-snapshot`; waits for completion |

## Known limitation — `restorePrehook` not yet triggered (Kasten ≤ 8.5.x)

> **Warning**: As of Kasten 8.5.x, the `restorePrehook` blueprint action is defined but
> **never triggered** by the Kasten executor during restore. Kasten engineering has confirmed
> this will be fixed in a near-future release. The `restorePrehook` is implemented in the
> blueprint so it will work automatically once the fix ships.
>
> Until then, **run the steps below manually before triggering a Kasten restore** for this
> namespace. Failing to do so will leave the ECK operator running while Kasten tries to
> replace the PVCs. ECK will immediately recreate pods, blocking PVC replacement and causing
> the restore to fail.

### Manual pre-restore steps (required until the fix ships)

Replace `elasticsearch-test` and `elasticsearch-sample` with your actual namespace and
Elasticsearch CR name:

```bash
# Pause ECK reconciliation so the operator stops managing the StatefulSets
kubectl annotate elasticsearches.elasticsearch.k8s.elastic.co elasticsearch-sample \
  -n elasticsearch-test common.k8s.elastic.co/pause=true --overwrite

# Scale down all Elasticsearch StatefulSets to release PVCs for replacement
kubectl scale statefulset -n elasticsearch-test \
  -l "elasticsearch.k8s.elastic.co/cluster-name=elasticsearch-sample" \
  --replicas=0

# Wait for all pods to terminate before triggering the Kasten restore
kubectl wait pod -n elasticsearch-test \
  -l "elasticsearch.k8s.elastic.co/cluster-name=elasticsearch-sample" \
  --for=delete --timeout=180s 2>/dev/null || true

echo "Ready — trigger the Kasten restore now."
```

Once these commands complete, trigger the Kasten restore. The `restorePosthook` will
remove the pause annotation and resume ECK reconciliation automatically.

---

## Prerequisites

### ECK operator

Install the ECK operator (CRDs + operator) in the `elastic-system` namespace:

```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.16.1/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.16.1/operator.yaml
kubectl -n elastic-system wait pod --all --for=condition=Ready --timeout=120s
```

### ReadWriteMany storage class with CSI snapshot support

The snapshot repository PVC requires a RWX storage class that Kasten can snapshot. Ensure:
- A RWX-capable CSI provisioner is installed on the cluster.
- A `VolumeSnapshotClass` exists for that provisioner.
- Kasten has a corresponding Retain-policy VolumeSnapshotClass registered.

Update the `storageClassName` in `operator/snapshot-repo-pvc.yaml` to match your environment.
The value used in this blueprint (`ebs-sc`) is the AWS EBS CSI class from the test environment.
Replace it with a RWX-capable CSI class that has a matching `VolumeSnapshotClass` registered
with Kasten (e.g. `azurefile-csi` on AKS, or an EFS-backed class on AWS).
Do **not** use legacy in-tree classes — they do not support CSI snapshots.

### Custom tool image

The blueprint uses `michaelcourcy/kasten-tools:8.5.2`, which adds `kubectl`, `tar`, `gzip` and  `jq`to
`gcr.io/kasten-images/kanister-tools:8.5.2` (the base image already contains `curl`).
See [images/kasten-tools/Dockerfile](images/kasten-tools/Dockerfile).

---

## Step 2 — Deploy the workload and create test data

### Deploy

```bash
kubectl create namespace elasticsearch-test

# Snapshot repository PVC (must exist before the Elasticsearch CR)
kubectl apply -f operator/snapshot-repo-pvc.yaml

# Elasticsearch cluster
kubectl apply -f operator/elasticsearch.yaml
```

Wait for the cluster to be ready (2–3 minutes):

```bash
kubectl -n elasticsearch-test get elasticsearch elasticsearch-sample -w
# Wait until HEALTH=green and PHASE=Ready
```

### Create test data

```bash
ES_PASSWORD=$(kubectl get secret elasticsearch-sample-es-elastic-user \
  -n elasticsearch-test -o jsonpath='{.data.elastic}' | base64 -d)

kubectl port-forward -n elasticsearch-test svc/elasticsearch-sample-es-http 9200:9200 &
PF_PID=$!
sleep 3

for i in 1 2 3; do
  curl -sk -u "elastic:${ES_PASSWORD}" -X POST \
    "https://localhost:9200/kasten-test/_doc" \
    -H 'Content-Type: application/json' \
    -d "{\"id\": $i, \"message\": \"kasten-backup-test-doc-$i\"}"
  echo
done

# Verify — expect "count" : 3
curl -sk -u "elastic:${ES_PASSWORD}" \
  "https://localhost:9200/kasten-test/_count?pretty"

kill $PF_PID
```

---

## Step 3 — Validate strategy manually (without blueprint)

### Register the repository and take a snapshot

```bash
ES_PASSWORD=$(kubectl get secret elasticsearch-sample-es-elastic-user \
  -n elasticsearch-test -o jsonpath='{.data.elastic}' | base64 -d)

kubectl port-forward -n elasticsearch-test svc/elasticsearch-sample-es-http 9200:9200 &
PF_PID=$!
sleep 3

# Register filesystem repository
curl -sk -u "elastic:${ES_PASSWORD}" -X PUT \
  "https://localhost:9200/_snapshot/kasten-repo?pretty" \
  -H 'Content-Type: application/json' \
  -d '{"type": "fs", "settings": {"location": "/mnt/snapshot-repo"}}'

# Delete previous snapshot if it exists (idempotency).
# Note: ?ignore_unavailable=true is NOT supported by the snapshot DELETE API in ES 8.x.
# Check the error type and only ignore snapshot_missing_exception (snapshot didn't exist yet).
DEL=$(curl -sk -u "elastic:${ES_PASSWORD}" -X DELETE \
  "https://localhost:9200/_snapshot/kasten-repo/kasten-snapshot")
ERR_TYPE=$(echo "${DEL}" | jq -r '.error.type // ""')
if [ -n "${ERR_TYPE}" ] && [ "${ERR_TYPE}" != "snapshot_missing_exception" ]; then
  echo "ERROR: ${DEL}"; exit 1
fi
echo "${DEL}"

# Take snapshot (synchronous — waits for completion)
curl -sk -u "elastic:${ES_PASSWORD}" -X PUT \
  "https://localhost:9200/_snapshot/kasten-repo/kasten-snapshot?wait_for_completion=true&pretty" \
  -H 'Content-Type: application/json' \
  -d '{"indices": "*", "ignore_unavailable": true, "include_global_state": false}'

kill $PF_PID
```

Expected: `"state" : "SUCCESS"`.

### CSI snapshot of the repo PVC (emulate Kasten)

```bash
# Replace the volumeSnapshotClassName with the one matching your RWX storage class
kubectl apply -f - <<EOF
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: snapshot-repo-manual-test
  namespace: elasticsearch-test
spec:
  volumeSnapshotClassName: csi-nfs-snapclass
  source:
    persistentVolumeClaimName: elasticsearch-snapshot-repo
EOF

kubectl wait volumesnapshot/snapshot-repo-manual-test \
  -n elasticsearch-test --for=jsonpath='{.status.readyToUse}'=true --timeout=120s
echo "Snapshot ready"
```

### Validate restore

```bash
ES_PASSWORD=$(kubectl get secret elasticsearch-sample-es-elastic-user \
  -n elasticsearch-test -o jsonpath='{.data.elastic}' | base64 -d)

kubectl port-forward -n elasticsearch-test svc/elasticsearch-sample-es-http 9200:9200 &
PF_PID=$!
sleep 3

# Simulate data loss
curl -sk -u "elastic:${ES_PASSWORD}" -X DELETE \
  "https://localhost:9200/kasten-test?pretty"

# Restore from the ES snapshot (repo PVC still mounted with the snapshot)
curl -sk -u "elastic:${ES_PASSWORD}" -X POST \
  "https://localhost:9200/_all/_close?pretty"

curl -sk -u "elastic:${ES_PASSWORD}" -X POST \
  "https://localhost:9200/_snapshot/kasten-repo/kasten-snapshot/_restore?wait_for_completion=true&pretty" \
  -H 'Content-Type: application/json' \
  -d '{"indices": "*", "ignore_unavailable": true, "include_global_state": false}'

# Verify — expect "count" : 3
curl -sk -u "elastic:${ES_PASSWORD}" \
  "https://localhost:9200/kasten-test/_count?pretty"

kill $PF_PID
```

---

## Step 4 — Deploy the blueprint

```bash
kubectl apply -f blueprint.yaml
kubectl apply -f blueprintbinding.yaml
```

---

## Step 5 — Test with Kasten

1. Create a backup policy on the `elasticsearch-test` namespace with a location profile.
2. Run an on-demand backup. Check kanister-svc logs for `backupPrehook`/`backupPosthook`.
3. Delete the `kasten-test` index (simulate data loss).
4. Restore from the restore point. Check executor logs for `restorePrehook`/`restorePosthook`.
5. Verify the restored data:

```bash
ES_PASSWORD=$(kubectl get secret elasticsearch-sample-es-elastic-user \
  -n elasticsearch-test -o jsonpath='{.data.elastic}' | base64 -d)
kubectl port-forward -n elasticsearch-test svc/elasticsearch-sample-es-http 9200:9200 &
PF_PID=$!; sleep 3
curl -sk -u "elastic:${ES_PASSWORD}" "https://localhost:9200/kasten-test/_count?pretty"
kill $PF_PID
```

```bash
# Debug logs
kubectl logs -n kasten-io -l app=kanister-svc --tail=200
kubectl logs -n kasten-io -l component=executor --tail=10000 -f
```

---

## Teardown

```bash
kubectl delete namespace elasticsearch-test

kubectl delete -f blueprintbinding.yaml
kubectl delete -f blueprint.yaml

kubectl delete restorepointcontent \
  -l k10.kasten.io/appNamespace=elasticsearch-test

# Uninstall ECK (if no longer needed)
kubectl delete -f https://download.elastic.co/downloads/eck/2.16.1/operator.yaml
kubectl delete -f https://download.elastic.co/downloads/eck/2.16.1/crds.yaml
kubectl delete namespace elastic-system
```
