# Kasten Blueprint — MariaDB Community Operator (Standalone)

## Tested versions

| Component | Version |
|---|---|
| Kubernetes | 1.32 (EKS) |
| Kasten | 8.5.2 |
| mariadb-operator (Helm chart) | 25.10.4 |
| MariaDB | 11.8.5 |

## Scope

This blueprint targets **standalone MariaDB instances** (replicas: 1) managed by the
[mariadb-operator](https://github.com/mariadb-operator/mariadb-operator) community operator.
Galera clusters and replicated topologies are out of scope.

## Chosen pattern: 2 — Quiesce / Unquiesce

### Rationale

| Pattern | Feasible? | Notes |
|---------|-----------|-------|
| 1 — Fence and quiesce a replica | ❌ | Standalone has no replica |
| 2 — Quiesce | ✅ **Chosen** | See below |
| 3/4 — Dump to temp PVC | Feasible | Ruled out: no incrementality |

**Why pattern 2:**

MariaDB exposes a standard quiesce primitive for InnoDB workloads:

```sql
FLUSH TABLES WITH READ LOCK;
FLUSH LOGS;
```

This blocks new writes and flushes all dirty pages and binary logs to disk, producing a
crash-consistent snapshot that InnoDB recovers cleanly on startup. After Kasten completes
the PVC snapshot, `UNLOCK TABLES` is issued in `backupPosthook` to resume normal operation.

Advantages:
- **Incremental**: Kasten snapshots the PVC delta between backups — backup time stays constant
  as the database grows.
- **No extra PVC**: No dump pod or temporary storage needed.
- **Simple restore**: Kasten restores the PVC; MariaDB starts up cleanly from the snapshot.
  The `FLUSH TABLES WITH READ LOCK` lock is a session-level in-memory state — it does not
  persist on disk. InnoDB sees a clean checkpoint and starts normally without any unlock step.
  A `restorePosthook` waits for the operator to reconcile the CR to `Ready` and then
  verifies the database accepts connections.

Limitation:
- Granular restore (single table or database) is not possible — the whole PVC is restored.
  If granular restore is required, switch to pattern 8 (logical dump on temp PVC).

## Operator ownership chain

```
MariaDB CR  (k8s.mariadb.com/v1alpha1)
  └── StatefulSet  (<name>)
        ├── Pod  (<name>-0)
        └── PVC  (storage-<name>-0)
```

Kasten discovers PVCs through the ownership chain from the MariaDB CR.
The blueprint is bound to the **MariaDB custom resource** via a BlueprintBinding.

## Blueprint actions

| Action | What it does |
|--------|-------------|
| `backupPrehook` | `FLUSH TABLES WITH READ LOCK; FLUSH LOGS;` — quiesces MariaDB |
| `backupPosthook` | `UNLOCK TABLES;` — unquiesces after Kasten PVC snapshot is ready |
| `restorePrehook` | Deletes the MariaDB CR so the operator stops reconciling, allowing Kasten to replace the PVC; waits for the StatefulSet to be garbage-collected before restore proceeds |
| `restorePosthook` | WaitV2 until the MariaDB CR reaches `Ready=True`; then runs `SELECT 1` inside the pod to confirm the database accepts connections |

## Known limitation — `restorePrehook` not yet triggered (Kasten ≤ 8.5.x)

> **Warning**: As of Kasten 8.5.x, the `restorePrehook` blueprint action is defined but
> **never triggered** by the Kasten executor during restore. Kasten engineering has confirmed
> this will be fixed in a near-future release. The `restorePrehook` is implemented in the
> blueprint so it will work automatically once the fix ships.
>
> Until then, **run the steps below manually before triggering a Kasten restore** for this
> namespace. Failing to do so will leave the MariaDB operator running while Kasten tries to
> replace the PVC. The operator will immediately recreate the PVC after Kasten deletes it,
> causing the restore to time out.

### Manual pre-restore steps (required until the fix ships)

Replace `mariadb-test` and `mariadb` with your actual namespace and MariaDB CR name:

```bash
# Delete the MariaDB CR so the operator stops reconciling the StatefulSet.
# Kasten will restore the CR manifest; the operator will then recreate the
# StatefulSet against the restored PVC.
kubectl delete mariadb mariadb -n mariadb-test --ignore-not-found=true

# Wait for the StatefulSet and pod to be fully deleted before triggering restore
kubectl wait statefulset/mariadb -n mariadb-test --for=delete --timeout=120s 2>/dev/null || true
kubectl wait pod/mariadb-0 -n mariadb-test --for=delete --timeout=120s 2>/dev/null || true

# Small grace period for PVC finalizer release
sleep 5
echo "Ready — trigger the Kasten restore now."
```

Once these commands complete, trigger the Kasten restore. After the restore point is applied,
Kasten restores the MariaDB CR manifest and the operator recreates the StatefulSet against
the restored PVC automatically.

---

## Dependencies

- mariadb-operator installed in the cluster (Helm chart `mariadb-operator/mariadb-operator`)
- A Kubernetes Secret holding the MariaDB root password (created by the operator by default)
- The blueprint and BlueprintBinding deployed in the `kasten-io` namespace
- Custom image `michaelcourcy/kasten-tools:<kasten-version>` for `KubeTask` phases that need
  `kubectl` (the standard `gcr.io/kasten-images/kanister-tools` image does not include it).
  Dockerfile: [`images/kasten-tools/Dockerfile`](images/kasten-tools/Dockerfile)

## Custom images

| Image | Base | Added | Dockerfile |
|-------|------|-------|------------|
| `michaelcourcy/kasten-tools:8.5.2` | `gcr.io/kasten-images/kanister-tools:8.5.2` | `kubectl` v1.32.0 | `images/kasten-tools/Dockerfile` |

To rebuild for a different Kasten version:
```bash
docker build --build-arg KASTEN_VERSION=<version> -t michaelcourcy/kasten-tools:<version> images/kasten-tools/
docker push michaelcourcy/kasten-tools:<version>
```

## Deploying the test workload

### 1 — Install mariadb-operator

```bash
helm repo add mariadb-operator https://helm.mariadb.com/mariadb-operator
helm repo update
helm install mariadb-operator mariadb-operator/mariadb-operator \
  --namespace mariadb-operator --create-namespace \
  --wait
```

### 2 — Deploy MariaDB

```bash
kubectl create namespace mariadb-test
kubectl apply -f operator/mariadb-root-secret.yaml
kubectl apply -f operator/mariadb.yaml
# Wait for the instance to be ready
kubectl wait mariadb/mariadb -n mariadb-test --for=condition=Ready --timeout=120s
```

### 3 — Create test data

```bash
kubectl exec -n mariadb-test mariadb-0 -c mariadb -- bash -c '
mariadb -uroot -p"${MARIADB_ROOT_PASSWORD}" -e "
  CREATE DATABASE IF NOT EXISTS test;
  USE test;
  CREATE TABLE IF NOT EXISTS employees (id INT, name VARCHAR(50), dept VARCHAR(50));
  INSERT INTO employees VALUES
    (1, '"'"'Alice'"'"', '"'"'Engineering'"'"'),
    (2, '"'"'Bob'"'"',   '"'"'Marketing'"'"'),
    (3, '"'"'Carol'"'"', '"'"'Engineering'"'"');
  SELECT * FROM employees;
"'
```

Expected output:
```
id  name   dept
1   Alice  Engineering
2   Bob    Marketing
3   Carol  Engineering
```

### 4 — Deploy the blueprint and binding

```bash
kubectl apply -f blueprint.yaml
kubectl apply -f blueprintbinding.yaml
```

### 5 — Verify the restore (after backup)

After a restore, re-run the SELECT to confirm data is recovered:

```bash
kubectl exec -n mariadb-test mariadb-0 -c mariadb -- bash -c '
mariadb -uroot -p"${MARIADB_ROOT_PASSWORD}" -e "USE test; SELECT * FROM employees;"'
```

## Removing the test workload

```bash
# Remove the blueprint and binding
kubectl delete -f blueprintbinding.yaml --ignore-not-found
kubectl delete -f blueprint.yaml --ignore-not-found

# Delete the MariaDB instance (operator garbage-collects the StatefulSet and pod;
# the PVC is retained by default — delete it explicitly)
kubectl delete -f operator/mariadb.yaml --ignore-not-found
kubectl delete pvc storage-mariadb-0 -n mariadb-test --ignore-not-found

# Delete the namespace and the root-password secret
kubectl delete namespace mariadb-test --ignore-not-found

# Uninstall mariadb-operator (only if no other MariaDB instances exist in the cluster)
helm uninstall mariadb-operator -n mariadb-operator
kubectl delete namespace mariadb-operator --ignore-not-found

# Delete the restorepoint content for this namespace 
kubectl delete restorepointcontent -l k10.kasten.io/appNamespace=mariadb-test
```

## Files

| File | Description |
|------|-------------|
| `operator/mariadb-root-secret.yaml` | Kubernetes Secret with the MariaDB root password |
| `operator/mariadb.yaml` | MariaDB CR for the test deployment |
| `blueprint.yaml` | Kasten Blueprint |
| `blueprintbinding.yaml` | BlueprintBinding targeting all MariaDB CRs |
| `images/kasten-tools/Dockerfile` | Dockerfile for `michaelcourcy/kasten-tools` |
