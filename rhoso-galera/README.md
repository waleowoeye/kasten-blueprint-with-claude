# RHOSO Galera — Kasten blueprint (Pattern 4: dump on a permanent Keeper PVC)

> ⚠️ **NOT YET VALIDATED.** This blueprint was built by speculation from the Red Hat
> KCS runbook *"How do I backup and restore galera clusters in Red Hat OpenStack
> Services on OpenShift"* and from the `openstack-k8s-operators/mariadb-operator`
> source code, **without access to a live RHOSO environment**. It is a starting
> point to walk through and validate with the customer. Every command marked
> "validate" below must be confirmed against a real cluster before production use.
> Step 2, 3 and 5 of the standard blueprint workflow (deploy / validate / E2E test)
> have **not** been performed.

Backs up the OpenStack databases held in a RHOSO 18 Galera cluster by producing a
logical `mysqldump` onto a dedicated **keeper PVC**, which Kasten snapshots. This
mirrors the Red Hat documented runbook exactly (`oc exec … mysqldump` /
`oc exec -i … mysql < dump`) — the keeper is simply "the workstation, but in-cluster
with a persistent PVC".

---

## Why Pattern 4

RHOSO's supported galera backup is a **logical dump**, not a volume snapshot of
`/var/lib/mysql`. A galera datadir is synchronously replicated and carries cluster
state (gcache, `grastate.dat`, GTID) that makes a raw single-volume snapshot an
unsupported restore path. Red Hat therefore dumps OpenStack schemas with
`mysqldump --single-transaction` from a pod *not currently serving traffic*.

Pattern 4 fits this perfectly:

- A dedicated **keeper Deployment** (one per galera cluster) mounts a **permanent
  PVC** at `/backup`. The PVC pre-exists at Kasten discovery time, so it is captured
  in the restore point.
- `backupPrehook` `KubeExec`s into the keeper, which `kubectl exec`s into the galera
  pod to run `mysqldump`, redirecting the output onto `/backup`. Kasten snapshots the
  keeper PVC.
- **BlueprintBinding** on the keeper (by label) preserves SRP/OCP — adding
  `openstack-cell1` is just another labelled keeper, no blueprint change.

The dump/reload run **inside the galera pod**, using that pod's own
`$DB_ROOT_PASSWORD` and TLS context. No credential is ever copied to the keeper or
into Kanister logs, and no remote-`root` grant or TLS client certificate is required.

### Patterns considered and ruled out

| Pattern | Ruled out because |
|---|---|
| 1 — Fence & quiesce a replica | A fenced-node datadir snapshot is not the supported galera restore path; the StatefulSet is operator-owned, so fencing/scaling it directly is reverted. |
| 2 — Quiesce + crash-consistent snapshot | Same: restoring a raw galera datadir is unsupported; Red Hat documents logical dump + reload. |
| 3 — Snapshot on a permanent *workload* PVC | Would require adding a backup volume to the operator-owned galera StatefulSet; the openstack-operator won't preserve arbitrary extra volumes, and the galera PVC is the datadir we're protecting. |
| 7–9 — Temporary PVC | Break BlueprintBinding (namespace-only context) and SRP/OCP; strictly worse here. |
| 11 — Vendor operator data mover | The operator has a `galerabackup` template but no stable external-immutability backup *API*; the documented customer method is manual `mysqldump`. |

---

## Versions

> Detect the Kasten version from your cluster rather than trusting this table.
> RHOSO is OLM-installed, so Kasten is usually installed by Helm in `kasten-io`:
> ```bash
> helm ls -n kasten-io
> # OpenShift OLM (if installed via OperatorHub):
> kubectl get csv -n kasten-io -o jsonpath='{.items[?(@.spec.displayName=="Kasten K10")].spec.version}'
> ```

| Component | Version (expected — **validate**) |
|---|---|
| OpenShift (RHOCP) | 4.16 / 4.18 (RHOSO 18 supported) |
| RHOSO | 18.0 (FR1) |
| mariadb-operator | [openstack-k8s-operators/mariadb-operator](https://github.com/openstack-k8s-operators/mariadb-operator) (bundled in RHOSO 18) |
| Galera / MariaDB | MariaDB 10.x / 11.x (per RHOSO 18 `openstack-mariadb` image) |
| Kasten | detect from cluster (e.g. `8.5.x`) |
| Keeper image | `registry.redhat.io/openshift4/ose-cli` (oc/kubectl) |

---

## How RHOSO Galera is laid out (facts this blueprint relies on)

Confirmed from the RHOSO mariadb-operator source
([github.com/openstack-k8s-operators/mariadb-operator](https://github.com/openstack-k8s-operators/mariadb-operator))
— **validate** on your cluster:

| Thing | Value |
|---|---|
| Galera CR | `galeras.mariadb.openstack.org/v1beta1`, e.g. `openstack`, `openstack-cell1` |
| StatefulSet | `<galera>-galera` (e.g. `openstack-galera`) |
| galera container | `galera` |
| data PVC | `mysql-db-<galera>-galera-<ordinal>` at `/var/lib/mysql` |
| active Service | `<galera>` — selector pins one pod via `statefulset.kubernetes.io/pod-name` |
| headless Service | `<galera>-galera` |
| data pods | `<galera>-galera-0..N` |
| root password (live) | secret named by `galera.status.rootDatabaseSecret`, key `DatabasePassword` (seeded from `osp-secret`/`DbRootPassword`) — **exposed in-pod as `$DB_ROOT_PASSWORD`** |
| scaling | **via `OpenStackControlPlane`** (`core.openstack.org/v1beta1`) `spec.galera.templates.<galera>.replicas` — editing the StatefulSet/Galera CR directly is reverted |
| per-service creds | `MariaDBAccount` CRs (`mariadb.openstack.org/v1beta1`) |
| image clients | galera image has `mysql`/`mysqldump`/`mysqladmin` + `python3`; **no `jq`** |

List the galera clusters in your control plane:

```bash
oc get galera --no-headers -o custom-columns=":metadata.name"
```

---

## Files

| File | Purpose |
|---|---|
| `keeper.yaml` | ServiceAccount + Role + RoleBinding, keeper PVC, keeper Deployment (one per galera cluster). |
| `blueprint.yaml` | The Kanister blueprint (`backupPrehook` / `backupPosthook` / `restorePrehook` / `restorePosthook`). |
| `blueprintbinding.yaml` | Binds the blueprint to every keeper by label `rhoso-galera-keeper: "true"`. |

---

## Blueprint actions

Kept in sync with `blueprint.yaml` at all times.

| Action | When Kasten calls it | What it does |
|---|---|---|
| `backupPrehook` | Before the keeper PVC snapshot | `KubeExec` into the keeper → find a galera pod *not* serving traffic (via the active-service selector + `wsrep_incoming_addresses`) → `mysqldump --single-transaction --databases` every non-system InnoDB schema into the single file `/backup/<galera>-databases.sql` (overwritten each run) → `sync`. |
| `backupPosthook` | After the keeper PVC snapshot is ready | `KubeExec` into the keeper → verify `/backup/<galera>-databases.sql` exists and is non-empty. |
| `restorePrehook` | Before PVC restore | **Not triggered in Kasten ≤ 8.5.x** (implemented for forward-compat). Scales galera to 1 via the OpenStackControlPlane and disables DB traffic (clears the active-service pod-name selector). |
| `restorePosthook` | After PVC restore | `KubeExec` into the keeper, in 3 phases: **(1)** idempotently ensure scaled-to-1 + traffic disabled; **(2)** drop existing OpenStack schemas and reload `/backup/<galera>-databases.sql` into `<galera>-galera-0`; **(3)** re-enable traffic and scale back to 3, then remind the operator to rotate credentials. |

---

## Prerequisites & deployment

The control plane lives in the `openstack` namespace; the keeper is deployed there too
(it must `kubectl exec` into the galera pods, which are in `openstack`).

### 1. Deploy a keeper per galera cluster

`keeper.yaml` provisions the keeper for galera `openstack`. **Set a CSI-snapshot-capable
storage class first** (see callout) and, for a second cluster, copy the PVC + Deployment
blocks renaming `openstack-keeper` → `openstack-cell1-keeper` (the `-keeper` suffix on the
keeper name is how the blueprint derives the galera name).

```bash
oc apply -f keeper.yaml
oc get deploy,pvc,sa -n openstack -l rhoso-galera-keeper=true
```

> **Scaling to many galera clusters (future convenience — not implemented yet):** because
> the only thing that varies between keepers is the name, it is trivial to wrap the PVC +
> Deployment in a small script (or Helm/Kustomize template) that takes the keeper name as
> an argument — e.g. `./deploy-keeper.sh openstack` / `./deploy-keeper.sh openstack-cell1` —
> and applies one keeper per `oc get galera` entry. The shared ServiceAccount/Role/RoleBinding
> can stay defined once. We are deliberately keeping `keeper.yaml` as plain, explicit
> manifests for now so the moving parts are obvious during validation; the templating
> wrapper can be added later without touching the blueprint or BlueprintBinding.

> **Storage class**: replace `<csi-snapshot-capable-storageclass>` in `keeper.yaml` with a
> storage class that supports CSI snapshots on your cluster and has a matching
> `VolumeSnapshotClass` registered with Kasten. On RHOSO this is commonly OpenShift Data
> Foundation (`ocs-storagecluster-ceph-rbd` with `ocs-storagecluster-rbdplugin-snapclass`)
> or LVMS. Do **not** use a class whose provisioner lacks CSI snapshot support.

> **RBAC**: `keeper.yaml` also creates the `rhoso-galera-keeper` ServiceAccount + Role +
> RoleBinding scoped to the `openstack` namespace (read+exec on galera pods; patch the
> active service; patch the OpenStackControlPlane; wait on the StatefulSet). These are a
> one-time prerequisite and are **not** part of the Kasten backup — they must exist
> independently for a restore to succeed. **Validate** the `core.openstack.org` /
> `mariadb.openstack.org` group names and that your security context permits this Role.

> **Image pull**: `registry.redhat.io/openshift4/ose-cli` is pulled via the cluster's
> global pull secret. If your keeper namespace cannot pull it, mirror the image or set an
> `imagePullSecret` on the `rhoso-galera-keeper` ServiceAccount.

#### Sizing the keeper PVC

OpenStack control-plane databases hold **metadata, not user data**, so they are small.
RHOSO 18 provisions each galera datadir at a default `storageRequest` of **`5000M`
(5 GB)** — that is Red Hat's design point for a normal control plane. The actual dataset
is usually a few hundred MB; busy or long-lived clouds drift into low single-digit GB.

The keeper PVC stores a **single logical `mysqldump`**, which is *smaller* than the
datadir (no secondary indexes, no InnoDB page overhead, no Galera `gcache`). The keeper
PVC therefore defaults to **`5Gi`** in `keeper.yaml` — mirroring the galera datadir
default, which is adequate-to-generous. Because the dump is overwritten in place each
backup (and Kasten keeps history in its restore points), the keeper PVC does **not** grow
with the number of backups — only with the size of the database itself.

What makes the database (and so the dump) grow:

- **Soft deletes** — Nova and Cinder keep deleted rows with a `deleted` flag rather than
  removing them, so tables accumulate indefinitely unless purged
  (`nova-manage db archive_deleted_rows` + `purge`, `cinder-manage db purge`). Neutron
  RBAC rules and large port/subnet counts add to it.
- Recommendation: **purge soft-deleted rows on a schedule.** This keeps both the galera
  datadir and the keeper PVC bounded.

Guidance:

- Default `5Gi` is fine for typical clouds. For large clouds with heavy soft-delete
  accumulation, raise the keeper PVC `storage` request (and consider purging first).
- A logical dump of a 5 GB datadir is typically a few hundred MB to ~1–2 GB. Streaming a
  dump of that size through `kubectl exec` completes in seconds to a few minutes — well
  within connection limits. The long-stream/timeout risk only becomes real for a cloud
  that has never purged soft-deletes; load-test on a realistic dump size before relying on
  it in production (this is the Step 2 "performance test" phase, out of scope here).

### 2. Install the blueprint + binding (in `kasten-io`)

```bash
oc apply -f blueprint.yaml
oc apply -f blueprintbinding.yaml
oc get blueprint,blueprintbinding -n kasten-io | grep rhoso-galera
```

### 3. Create a backup policy scoped to the keeper(s)

Back up **only the keeper Deployment + its PVC** — *not* the whole control-plane
namespace. The keeper PVC carries the dump; the galera datadir PVCs must **not** be
restored (that would be the unsupported raw-datadir restore). Selecting the keeper by
label includes its PVC and excludes the operator-owned galera StatefulSet/PVCs.

- Scope: namespace `openstack`, filtered to objects with label
  `rhoso-galera-keeper=true`.
- The policy's `backupParameters.profile` must reference a **`Location`** profile (S3/GCS/
  Azure), not an `Infra` profile, so Kanister hooks can run during backup and restore.
- An export location profile is **not** required for this pattern.

> **Scope & disaster recovery — explicit assumption.** This policy deliberately captures
> **only the keeper artifact (the logical dump)**, not the `Galera` CR, the
> `OpenStackControlPlane`, or any other OpenStack control-plane resource. The benefits are
> that backups are fast, restores are simple, and recovery is naturally **granular** —
> restoring a single keeper restores a single galera cluster's data, independently of the
> others.
>
> The trade-off is that this blueprint **assumes the galera clusters (and the wider
> control plane) already exist, or will be recreated, by a GitOps process** such as Argo CD
> or Flux that owns the `OpenStackControlPlane` / `Galera` CRs as declarative source of
> truth. In a full disaster-recovery scenario the sequence is therefore:
> **(1)** GitOps reconciles the control plane and brings the galera clusters back up empty,
> then **(2)** this blueprint's restore reloads the OpenStack data into them from the keeper
> dump. Kasten is responsible for the *data*, GitOps for the *infrastructure* — Kasten should
> not be the thing that recreates the operators or the control plane itself.

Trigger an on-demand run (RunAction / "run once" in the UI) and confirm:

```bash
# Kanister controller / executor logs
oc logs -n kasten-io -l component=executor --tail=10000 -f
```

Expect `galeraDump` to select a non-serving pod, dump the schemas, and `verifyDump` to
report a non-empty dump. The RunAction status must be `Complete`.

---

## Restore

> ⚠️ **`restorePrehook` is not triggered in Kasten ≤ 8.5.x.** The blueprint therefore
> performs the required pre-reload prep **idempotently at the start of `restorePosthook`**,
> so restore works today. When the fix ships, `restorePrehook` will run the same prep
> earlier and the posthook prep becomes a no-op. No manual step is needed for this prep on
> current Kasten — but **do** follow the credential-rotation runbook below.

### Automated by the blueprint (`restorePosthook`)

`RestorePoint` and `RestoreAction` live in the **application namespace** (`openstack`). You
do **not** need to specify a profile in the `RestoreAction` — Kasten extracts the location
profile required for the Kanister hook from the `RestorePointContent` automatically.

```bash
oc get restorepoint -n openstack

oc create -f - --validate=false <<'EOF'
apiVersion: actions.kio.kasten.io/v1alpha1
kind: RestoreAction
metadata:
  generateName: restore-
  namespace: openstack
spec:
  subject:
    apiVersion: apps.kio.kasten.io/v1alpha1
    kind: RestorePoint
    name: <RESTORE_POINT_NAME>          # from the command above
    namespace: openstack
  targetNamespace: openstack
EOF
```

`restorePosthook` then: scales galera to 1 (via the OpenStackControlPlane) → disables DB
traffic → drops existing OpenStack schemas → reloads the dump into `<galera>-galera-0` →
re-enables traffic → scales back to 3.

### Manual fallback (if the hook is unavailable / for dry runs)

This is the Red Hat runbook the hook automates. `$DB_ROOT_PASSWORD` is available inside
the galera container.

```bash
DBNAME=openstack
CONTROLPLANE=$(oc get oscp -n openstack -o jsonpath='{.items[0].metadata.name}')
BACKUP_FILE=/backup/${DBNAME}-databases.sql

# Scale to 1 pod
oc patch oscp $CONTROLPLANE -n openstack --type merge \
  -p '{"spec":{"galera":{"templates":{"'$DBNAME'":{"replicas":1}}}}}'
oc wait sts $DBNAME-galera -n openstack --for=jsonpath='{.status.availableReplicas}'=1

# Disable DB traffic
oc patch svc $DBNAME -n openstack --type=merge \
  -p '{"spec":{"selector":{"statefulset.kubernetes.io/pod-name":""}}}'

# (optional) drop existing OpenStack schemas
oc exec -q -c galera $DBNAME-galera-0 -n openstack -- bash -c \
  "mysql -uroot -p\$DB_ROOT_PASSWORD -sN -e \"select distinct table_schema from information_schema.tables where engine='innodb' and table_schema != 'mysql';\" | xargs -r -n1 mysqladmin -uroot -p\$DB_ROOT_PASSWORD -f drop"

# Reload (run from the keeper, where /backup is mounted)
oc exec -q -i -c keeper deploy/$DBNAME-keeper -n openstack -- \
  bash -c "cat $BACKUP_FILE" | \
  oc exec -q -i -c galera $DBNAME-galera-0 -n openstack -- bash -c "mysql -uroot -p\$DB_ROOT_PASSWORD"

# Re-enable traffic + scale back to 3
oc patch svc $DBNAME -n openstack --type=merge \
  -p '{"spec":{"selector":{"statefulset.kubernetes.io/pod-name":"'$DBNAME-galera-0'"}}}'
oc patch oscp $CONTROLPLANE -n openstack --type merge \
  -p '{"spec":{"galera":{"templates":{"'$DBNAME'":{"replicas":3}}}}}'
oc wait sts $DBNAME-galera -n openstack --for=jsonpath='{.status.availableReplicas}'=3
```

### Restore OpenStack credentials (MANUAL — required after every restore)

The reloaded dump restores **schemas and data**, but the OpenStack service credentials in
the database must be re-synchronised with the running services. Per the Red Hat runbook
you must perform a full **MariaDBAccount** password rotation for every OpenStack service.
This is intentionally **not** automated by the blueprint: it triggers rolling restarts of
OpenStack services and is specific to your control-plane topology.

High-level rotation workflow (see the Red Hat docs for the exact per-service steps):

1. Create a **new `MariaDBAccount` CR** for each OpenStack service (creates new DB
   credentials without deleting the existing ones).
2. Reference the new `MariaDBAccount` in each service's CR — this triggers a rolling
   restart of that service.
3. Once all services are restarted, delete the old `MariaDBAccount` CRs.

### Assumptions on why Red Hat asks to execute a credential rotation

> The Red Hat note states the *requirement* but not the *mechanism*. The following is our
> **working hypothesis** to support the customer conversation — it is reasoned from two
> facts we are confident about, not a verbatim Red Hat explanation. Confirm against the
> `mariadb-operator` reconcile code / Red Hat support before treating it as definitive.

**Two facts that drive it:**

1. **The backup excludes the `mysql` system schema.** The dump query is
   `... where engine='innodb' and table_schema != 'mysql'`, so the dump contains only the
   OpenStack *application* schemas (keystone, nova, neutron, cinder, …). The **database
   user accounts, their password hashes, and their `GRANT`s are NOT in the backup** —
   those live in the `mysql` schema, which is excluded.
2. **A `MariaDBAccount` is a *database-level* credential** — the user + password an
   OpenStack service uses to *connect to MariaDB* (not a Keystone/API credential). The
   mariadb-operator reconciles each `MariaDBAccount` by ensuring that DB user exists, holds
   the right grants on its database, and carries the password stored in its Kubernetes
   Secret.

**Why a restore can desynchronise credentials:**

The restore does **drop + reload** the application databases (`mysqladmin … drop`, then
replay the dump). Consequences:

- `DROP DATABASE <svc>` removes the grants service users held **on that database**. The
  dump's `CREATE DATABASE`/table statements recreate the schema but carry **no `GRANT`
  statements** (grants live in the excluded `mysql` schema). So after reload a service's DB
  user may still exist but no longer have privileges on the freshly-recreated database →
  the service authenticates but hits *"access denied"* on queries.
- More broadly, the restored databases are now at a point-in-time state that is not
  guaranteed to line up with the user/grant state the running cluster currently holds.

**Why rotation is the prescribed fix (and why it's zero-downtime):**

Rather than hand-patching grants, creating a **fresh `MariaDBAccount` per service** makes
the operator deterministically re-provision the user, **re-establish the grants on the
restored database, and set a known password**, then roll the service to pick it up. The
documented three-step flow — *create new → repoint service (rolling restart) → delete old*
— is exactly a zero-downtime credential rotation: every service ends up with a
user/grant/password triple that is internally consistent with the just-restored databases,
without an outage.

**One-sentence summary for the customer:** because the backup excludes the grant tables and
the restore drops/recreates the databases, the service↔database credentials and grants can
drift, and rotating the `MariaDBAccount`s is the operator-native way to re-sync them
cleanly.

> **Open question to confirm:** whether the drift is purely about grants (the strongest
> hypothesis above) or also involves the password hash in `mysql.user` becoming stale
> relative to the Kubernetes Secret. Verifying how the operator applies `GRANT` /
> `SET PASSWORD` for a `MariaDBAccount` would settle this.

---

## Cleanup

```bash
# Remove blueprint + binding
oc delete -f blueprintbinding.yaml -f blueprint.yaml

# Remove the keeper(s) and RBAC (repeat/adjust for openstack-cell1-keeper)
oc delete -f keeper.yaml

# Remove restore point content created during testing
oc delete restorepointcontent -l k10.kasten.io/appNamespace=openstack
```

> Deleting `keeper.yaml` removes the keeper PVC and its dumps. The galera clusters and
> their data are untouched — this blueprint never owns or deletes galera datadir PVCs.

---

## Open items to validate with the customer

1. **`$DB_ROOT_PASSWORD` in `oc exec`** — the Red Hat runbook relies on it being present in
   an exec'd galera-container shell. Confirm on the actual image/version.
2. **`core.openstack.org` / `mariadb.openstack.org` group + resource names** and that the
   keeper Role is permitted under the cluster's security policies.
3. **OpenStackControlPlane scaling** of an individual galera template (`replicas: 1`/`3`)
   and that the openstack-operator honours it without side effects on other services.
4. **Active-service selector toggling** (`statefulset.kubernetes.io/pod-name`) behaves as
   documented for disabling/enabling traffic.
5. **`kubectl wait --for=jsonpath`** is supported by the chosen `ose-cli` version.
6. **Backup-policy scoping** correctly captures only the keeper PVC and excludes the galera
   datadir PVCs.
7. **Multiple galera clusters** — confirm the per-keeper naming convention and that one
   policy over the `rhoso-galera-keeper=true` label backs them all up correctly.
8. **Credential rotation runbook** — map it to the customer's exact set of OpenStack
   services.
