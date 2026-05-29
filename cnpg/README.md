# CNPG (CloudNativePG) — Kasten Blueprint

Backup and restore for [CloudNativePG](https://cloudnative-pg.io/) PostgreSQL clusters using
**Pattern 1: Fence and quiesce a replica**.

## Versions

| Component | Version |
|---|---|
| Kubernetes | `1.32` (EKS) |
| Kasten | `8.5.4` |
| CNPG Helm chart | `cloudnative-pg 0.28.0` |
| CNPG operator | `1.29.0` |
| PostgreSQL | `18.3` |

## Pattern

**Pattern 1 — Fence and quiesce a replica.**
A standby replica's WAL replay is paused immediately before Kasten snapshots the PVCs.
This creates a guaranteed-consistent checkpoint on the replica with zero primary impact.
The quiesced replica's PVC snapshot is the sole source of truth for restore.

## How it works

### Backup

1. `backupPrehook` pauses WAL replay on a replica pod (`pg_wal_replay_pause()`).
2. Kasten snapshots **only the replica PVC** (configured via policy resource filter — see below).
3. `backupPosthook` resumes WAL replay (`pg_wal_replay_resume()`).

### Restore

1. `restorePrehook` deletes the CNPG Cluster CR. The operator stops all pods and
   garbage-collects PVCs (CNPG owns PVC lifecycle).
2. Kasten restores the replica PVC from the quiesced snapshot, with its original CNPG labels.
3. Kasten recreates the Cluster CR from the backup.
4. The CNPG operator detects the existing replica PVC with data (`latestGeneratedNode > 0`),
   **promotes it to primary automatically** — no `initdb` is run.
5. A fresh replica is created via streaming replication.
6. `restorePosthook` waits for the cluster condition `Ready`.

## Blueprint actions

| Action | What it does |
|---|---|
| `backupPrehook` | Finds a replica pod, pauses WAL replay (`pg_wal_replay_pause()`) |
| `backupPosthook` | Resumes WAL replay on the same replica (`pg_wal_replay_resume()`) |
| `restorePrehook` | Deletes the Cluster CR so the operator stops all pods and PVCs before Kasten restores (**not yet triggered in Kasten ≤ 8.5.x — see warning below**) |
| `restorePosthook` | Waits for the cluster condition `Ready` (operator handles primary promotion automatically) |

## ⚠️ restorePrehook — manual workaround (Kasten ≤ 8.5.x)

`restorePrehook` is **not yet triggered** by Kasten in versions ≤ 8.5.x. Until the fix ships,
perform these steps **manually before triggering a Kasten restore**:

```bash
# Delete the Cluster CR — operator will terminate all pods and remove PVCs
kubectl delete cluster.postgresql.cnpg.io pg-cluster -n cnpg-test

# Confirm all pods and PVCs are gone
kubectl get pods -n cnpg-test
kubectl get pvc -n cnpg-test
```

Once the restore completes, `restorePosthook` waits for the cluster to be ready automatically.

## Kasten policy — PVC resource filter (required)

The backup policy **must** exclude the primary PVC so only the quiesced replica PVC is snapshotted.
Use `excludeResources` in `spec.actions[].backupParameters.filters`:

```yaml
apiVersion: config.kio.kasten.io/v1alpha1
kind: Policy
metadata:
  name: cnpg-backup-policy
  namespace: cnpg-test
spec:
  comment: "CNPG backup - quiesced replica PVC only, primary excluded"
  frequency: "@onDemand"
  actions:
    - action: backup
      backupParameters:
        profile:
          name: <LOCATION_PROFILE>
          namespace: kasten-io
        filters:
          excludeResources:
            - matchLabels:
                cnpg.io/instanceRole: primary
  selector:
    matchExpressions:
      - key: k10.kasten.io/appNamespace
        operator: In
        values:
          - <APP_NAMESPACE>
```

This ensures Kasten snapshots only the quiesced replica PVC.
When triggering a RestoreAction, Kasten will restore the replica PVC only — the CNPG operator
then auto-promotes it to primary.

## Prerequisites

### Install the CNPG operator

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update cnpg
helm install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system \
  --create-namespace \
  --version 0.28.0 \
  --wait --timeout 3m
```

### Deploy the blueprint and binding

```bash
kubectl apply -f blueprint.yaml
kubectl apply -f blueprintbinding.yaml
```

## Deploy the test workload

```bash
kubectl create namespace cnpg-test

cat <<EOF | kubectl apply -f -
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: pg-cluster
  namespace: cnpg-test
spec:
  instances: 2
  storage:
    size: 1Gi
    storageClass: ebs-sc
EOF
```

> **Storage class**: `ebs-sc` is the AWS EBS CSI storage class used in our test environment.
> Replace it with a storage class that supports CSI snapshots on your cluster
> (e.g. `managed-csi` on AKS, `standard-rwo` on GKE, your custom class on bare-metal, etc.).
> The class must have a matching `VolumeSnapshotClass` registered with Kasten.
> Do **not** use legacy in-tree classes (e.g. `gp2` on AWS) — they do not support CSI snapshots.

Wait for the cluster to be ready:

```bash
kubectl wait cluster.postgresql.cnpg.io pg-cluster -n cnpg-test \
  --for=condition=Ready --timeout=8m
kubectl get pods -n cnpg-test -l cnpg.io/cluster=pg-cluster
```

## Create test data

```bash
kubectl exec -n cnpg-test pg-cluster-1 -- \
  psql -U postgres -c "CREATE DATABASE kasten_test;"

kubectl exec -n cnpg-test pg-cluster-1 -- \
  psql -U postgres -d kasten_test -c "
    CREATE TABLE employees (
      id SERIAL PRIMARY KEY,
      name VARCHAR(50),
      department VARCHAR(50),
      salary INTEGER
    );
    INSERT INTO employees (name, department, salary) VALUES
      ('Alice Martin',   'Engineering', 95000),
      ('Bob Chen',       'Engineering', 88000),
      ('Carol Smith',    'Marketing',   72000),
      ('David Johnson',  'Sales',       68000),
      ('Eve Williams',   'Engineering', 102000);
  "

kubectl exec -n cnpg-test pg-cluster-1 -- \
  psql -U postgres -d kasten_test -c "SELECT * FROM employees ORDER BY id;"
```

Run the policy and corrupt some data :

```bash 

kubectl exec -n cnpg-test pg-cluster-1 -- \
  psql -U postgres -d kasten_test -c "DELETE FROM employees WHERE department = 'Engineering';"

kubectl exec -n cnpg-test pg-cluster-1 -- \
  psql -U postgres -d kasten_test -c "SELECT * FROM employees ORDER BY id;"

```

then create a restore action from the last restore point 

```bash
# delete the cnpg cluster 
kubectl delete clusters.postgresql.cnpg.io pg-cluster -n cnpg-test

RESTORE_POINT=$(kubectl get restorepoint -n cnpg-test \
  -o jsonpath='{.items[-1].metadata.name}')

kubectl create -f - <<EOF
apiVersion: actions.kio.kasten.io/v1alpha1
kind: RestoreAction
metadata:
  generateName: restore-
  namespace: cnpg-test
spec:
  subject:
    apiVersion: apps.kio.kasten.io/v1alpha1
    kind: RestorePoint
    name: ${RESTORE_POINT}
    namespace: cnpg-test
  targetNamespace: cnpg-test
EOF
```

> No `profile` is needed in the `RestoreAction` — Kasten extracts the location profile from
> the `RestorePointContent` automatically.



## Validate data after restore


```bash
# The primary pod name changes after restore (CNPG promotes the replica)
PRIMARY=$(kubectl get pods -n cnpg-test \
  -l "cnpg.io/cluster=pg-cluster,cnpg.io/instanceRole=primary" \
  -o jsonpath='{.items[0].metadata.name}')

kubectl exec -n cnpg-test "$PRIMARY" -- \
  psql -U postgres -d kasten_test -c "SELECT * FROM employees ORDER BY id;"
# Expected: 5 rows — Alice, Bob, Carol, David, Eve
```

## Destroy the test workload

```bash
kubectl delete namespace cnpg-test

# Clean up restore point contents created by Kasten
kubectl delete restorepointcontent -l k10.kasten.io/appNamespace=cnpg-test
```

## Uninstall the operator and blueprint

```bash
helm uninstall cnpg -n cnpg-system
kubectl delete namespace cnpg-system
kubectl delete blueprint cnpg-blueprint -n kasten-io
kubectl delete blueprintbinding cnpg-blueprint-binding -n kasten-io
```
