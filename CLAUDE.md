# Kasten Blueprint Development — Rules and Patterns

> Full mechanics, YAML examples, and template reference: see [kasten-kanister.md](kasten-kanister.md)

## Core principle

**Kasten is the datamover.** Blueprints must not transfer data themselves. Kasten snapshots the PVCs attached to the resource. Blueprints only handle surrounding logic: quiescing, dump-to-PVC when needed, and unquiescing.

Never use in Kasten blueprints: `KanisterBackupData`, `KanisterRestoreData`, `KanisterBackupDataDelete`, `kopiaSnapshot` artifacts.

---

## Reserved action names (annotation / BlueprintBinding mode)

Kasten only recognises these six names. Any other action name is silently ignored.

| Action | When called | Use for |
|---|---|---|
| `backupPrehook` | Before PVC snapshots are initiated| Quiesce, or create dump PVC |
| `backup` | Blueprint manages data movement | **Avoid** — use PVC snapshots |
| `backupPosthook` | After PVC snapshots are ready | Unquiesce, or delete dump PVC |
| `restorePrehook` | Before PVC restore | Pre-restore preparation (**not yet triggered** in Kasten ≤ 8.5.x — always implement for when the fix ships; document manual steps in `README.md`) |
| `restore` | Blueprint manages data movement | **Avoid** |
| `restorePosthook` | After PVC restore | Post-restore init, cache warming |

For action hooks (namespace context), any action name is valid — convention is the same names above. See [kasten-kanister.md](kasten-kanister.md#invocation-mode-2--action-hooks).

---

## Blueprint development workflow

Follow these five steps in order when developing a new blueprint.

### Step 1 — Choose a pattern and document it

Select the backup pattern from the list in [kasten-kanister.md](kasten-kanister.md#blueprint-patterns). For workloads where **Kasten is the data mover**, apply them in this preferred order — stop at the first one that is technically feasible for the target database:

**Permanent PVC patterns — BlueprintBinding supported:**

1. **Fence and quiesce a replica** — best choice when a replica exists; zero primary impact.
2. **Quiesce** — preferred when crash-consistent snapshots are sufficient and incrementality matters (backup time stays constant as data grows).
3. **Database snapshot on a permanent workload PVC (PVC mounted by workload)** — the workload mounts a dedicated backup PVC; `backupPrehook` uses `KubeExec` to run the snapshot tool in the workload pod. PVC pre-exists at discovery time. BlueprintBinding on the workload CR.
4. **Database dump or snapshot on a permanent Keeper PVC (PVC mounted by keeper)** — a dedicated Deployment that runs the backup tool image mounts the backup PVC permanently; `backupPrehook` uses `KubeExec` into the keeper pod. PVC pre-exists at discovery time. BlueprintBinding on the keeper Deployment.
5. **Database dump or snapshot via a local MinIO keeper (PVC mounted by a keeper, but the keeper exposes S3-protocol)** — a MinIO Deployment in the application namespace acts as the local S3 endpoint, backed by a permanent PVC; the workload's native S3 backup tool writes to MinIO and Kasten snapshots the MinIO PVC. BlueprintBinding on the MinIO Deployment.
6. **Create logical dump or database snapshot on PVC already used by the database** — when no extra PVC is acceptable and the dump can safely coexist on the database volumes without exhausting storage. Dangerous; an extra PVC should always be preferred when possible.

> **Why permanent PVC patterns are ranked first:** Kasten's PVC discovery runs **before**
> `backupPrehook` executes. Permanent PVCs pre-exist at discovery time and are correctly
> snapshotted. They work with BlueprintBinding, which gives each workload type its own blueprint
> and preserves the **Single Responsibility Principle (SRP)** and **Open/Closed Principle (OCP)**:
> adding a new workload type never requires modifying an existing blueprint.

**Temporary PVC patterns — BackupAction preHook required, BlueprintBinding NOT supported:**

> ⚠️ Any PVC created during `backupPrehook` will **not** be included in the restore point,
> because PVC discovery runs before `backupPrehook`. The only way to include a temporary PVC
> in a restore point is to create it inside a **BackupAction preHook** (action hook), which runs
> before PVC discovery. This means:
> - Context is namespace-only (`{{ .Namespace.Name }}` only — no object-level fields).
> - One action hook blueprint covers all workloads in the namespace — a **SRP violation**.
> - Adding a second workload type requiring a temporary PVC requires modifying the same blueprint —
>   an **OCP violation**.
> - **Only deploy one type of workload per namespace** when using these patterns.
> Only choose patterns 7–9 when patterns 1–6 are technically infeasible.

7. **Database snapshot on a temporary PVC** — database snapshot tool writes to a temporary PVC created in a BackupAction preHook before discovery. Incrementality via append-only snapshot files.
8. **Logical dump on a temporary PVC** — dump tool writes to a temporary PVC created in a BackupAction preHook. No incrementality.
9. **Create dump of an external database on a temporary PVC** — for databases outside the cluster whose dump can be created inside the cluster. No incrementality.

When **Kasten is not the data mover**, two additional patterns apply. These are not ranked against the list above — they are the only valid options when the database is managed or when the operator owns the backup mechanism:

10. **Trigger a dump of a managed database** — for cloud-managed databases (RDS, Firestore, MongoDB Atlas, etc.); the cloud provider owns immutability.
11. **Use vendor operator data mover** — when the operator has a built-in backup API (CNPG, K8ssandra Medusa, Crunchy Postgres, etc.); the operator owns immutability.

The choice must balance implementation complexity (backup **and** restore), and whether incrementality is critical (patterns 1–5 with append-only snapshots are incremental; plain dump patterns are not).

> ⛔ **MANDATORY STOP — do not proceed to Step 2 until the user has explicitly approved the pattern.**
>
> When you reach this point you MUST:
> 1. State the pattern you have chosen and explain why it is the best fit for this workload.
> 2. List any patterns you considered and ruled out, with one-line reasons.
> 3. Ask the user explicitly: *"Do you agree with this pattern choice? Should I proceed to Step 2?"*
> 4. **Wait for the user's reply.** Do not write any README, deploy any workload, or write any code
>    until the user confirms. If the user asks for changes, revise the analysis and repeat from point 1.
>
> Writing the README is part of Step 2, not Step 1. The only output of Step 1 is the user's
> confirmed approval of the pattern.

### Step 2 — Deploy the workload and create test data

Deploy the target application in a test namespace. Create a representative minimum dataset that will let you validate a restore (e.g. a known set of records, files, or keys that you can verify after recovery). For the speed of the development lifecycle this is very important to create a limited dataset (Under 5Kb). Later once the whole blueprint works you can try performance test by increasing the size of the dataset. But this is out of the scope of this activity. 

The way you deploy the workload and how you create test data should be documented in the README.md so that
user can test it on their own. Also you must describe how to remove the workload and its dependencies.

**Storage class in README examples**: whenever a `storageClassName` appears in a README deployment example, add a callout note immediately after the manifest reminding the reader that the value is cluster-specific and must be replaced with a CSI storage class that supports snapshots on their cluster. Example note:

> **Storage class**: `ebs-sc` is the AWS EBS CSI storage class used in our test environment.
> Replace it with a storage class that supports CSI snapshots on your cluster
> (e.g. `managed-csi` on AKS, `standard-rwo` on GKE, your custom class on bare-metal, etc.).
> The class must have a matching `VolumeSnapshotClass` registered with Kasten.
> Do **not** use legacy in-tree classes (e.g. `gp2` on AWS) — they do not support CSI snapshots.

Describe also the deletion of the restorepointcontent created for the clean up :
```
kubectl delete restorepointcontent -l k10.kasten.io/appNamespace=<NAMESPACE USED FOR THE TEST>
```

### Step 3 — Validate the backup/restore workflow without a blueprint

The goal of this step is to fully validate the chosen backup/restore workflow for the target workload, **without the complexity of a blueprint or Kasten policy**. Step 4 then automates exactly what was validated here. These are two separate concerns: first prove the workflow works, then encode it.

Execute each operation using `kubectl exec`, `kubectl cp`, or direct API calls. Use these primitives to emulate what Kasten does at each stage:
- **PVC backup**: use a CSI snapshot to emulate what Kasten does to application PVCs. Follow [this example](https://github.com/michaelcourcy/test-csi-snapshot) if you don't know how to create csi snapshot. Always make sure that you wait for snapshots to be ready before you test the backupPosthook.
- **Object storage**: deploy a MinIO instance in a dedicated namespace to emulate an S3 endpoint (needed for *Use vendor operator data mover* patterns). You can find an example for deploying minio [here](https://github.com/michaelcourcy/kasten-s3troubleshooting?tab=readme-ov-file#test-with-a-minio-instance).
- **Custom image**: if the backup/restore commands require tools not available in a standard base image, build and push a custom image at this step (see [The dump-tool image](kasten-kanister.md#the-dump-tool-image)). The image is needed here to validate that the commands execute correctly in a Kubernetes context — as a sidecar of the workload, or as a standalone pod attached or not to a PVC depending on the pattern. Do not defer image building to Step 4: if the image is wrong, Step 3 will catch it before any blueprint is written.

Verify that:
- Deleting the test data after a backup and then restoring it retrieves the expected records.

### Step 4 — Implement the blueprint and document dependencies

Write the Blueprint (and BlueprintBinding if needed). Document all deployment prerequisites in the `README.md`:
- Any custom dump image (base image, tools added, registry location).
- Extra operator configuration required (e.g. enabling a backup API, creating a backup user).
- Extra Helm values or manifests that must be applied before the blueprint works.

**Always include a versions table** in the `README.md` documenting the exact versions used when the blueprint was developed and tested. Blueprint behaviour is often version-sensitive (operator APIs change, database backup semantics differ across releases). The table must cover at minimum:

| Component | Version |
|---|---|
| Kubernetes | e.g. `1.32` (EKS) |
| Kasten | e.g. `8.5.4` |
| Operator / Helm chart | e.g. `mariadb-operator 25.10.4` |
| Database / Application | e.g. `MariaDB 11.8.5` |

**Always detect the Kasten version from the cluster** rather than hardcoding it. Use the appropriate command for the deployment method:

```bash
# Helm (most common)
helm ls -n kasten-io

# OpenShift OLM (CSV)
kubectl get csv -n kasten-io -o jsonpath='{.items[?(@.spec.displayName=="Kasten K10")].spec.version}'
```

Omit the operator row for workloads deployed without an operator (plain Helm chart only).

**`restorePrehook` is not yet triggered in Kasten ≤ 8.5.x.** Always implement it in the blueprint (for forward compatibility), and add a prominent warning section in the `README.md` with the equivalent manual steps the operator must run before triggering a Kasten restore until the fix ships.

**Keep the "Blueprint actions" table in `README.md` in sync with the blueprint YAML at all times.**
Every action defined in the blueprint (`backupPrehook`, `backupPosthook`, `restorePrehook`, `restorePosthook`, etc.) must have a row in that table describing what it does. No action may exist in the YAML without a corresponding row, and no row may exist without a corresponding action in the YAML.

If a custom container image is needed (e.g. a tool image for `KubeTask` or a kubeOps that create a pod), commit its `Dockerfile`
in a subdirectory of the blueprint folder: `<blueprint-dir>/images/<image-name>/Dockerfile`.
**Never reference a custom image in a blueprint without a committed Dockerfile.** Document the
image name, base image, and what was added in the `README.md`.

Do not assume that the base image contains jq, yq or kubectl, most of the time you have to add them.

### Step 5 — Test the blueprint end-to-end

Run a full backup/restore cycle through Kasten:
- Ensure the workload is bound through the kasten annotation, a blueprintbinding or an action hook
- Create a backup policy on demand with location profile for Kanister actions (Export Location Profile
is optional and useless in this context) and trigger a runaction.

**Never create a Kanister ActionSet directly to test a blueprint.** Kasten sets up the ActionSet
context differently from a manually created one — for example, `.Namespace.Name` is only populated
when Kasten invokes an action hook, not in a hand-crafted ActionSet. A manual ActionSet will
silently resolve template variables to empty strings, causing incorrect behaviour. Always trigger
testing through a Kasten RunAction.
- Check in the kanister logs that the blueprint executed as expected, check the status of the runaction is completed
- Corrupt or delete the test data.
- Restore from the restore point and verify that:
    1. kanister logs show expected output
    2. the status of the restore action is completed 
    3. the test data is recovered correctly.

Iterate until success without coming back to step 1 unless you discover step 1 can not be implemented.

**If Step 3 succeeds but Step 5 fails, do not change the pattern.** Step 5 is purely the test of the automation of Step 3 — if the blueprint does not behave as the validated workflow did, there is a blueprint or Kasten integration issue, not a strategy issue. Stop, explain the discrepancy to the user, and work through it together before considering any change of pattern.


---

## Deployment rules

- Always deploy blueprints in the **`kasten-io` namespace**.
- **Separate operator namespace from workload namespace.** Deploy Kubernetes operators (PSMDB,
  CNPG, Couchbase, etc.) in a dedicated namespace (e.g. `psmdb-operator`) and workload clusters
  in separate namespaces (e.g. `psmdb-test`). Use `watchAllNamespaces=true` or equivalent so the
  operator manages workloads across namespaces. This keeps operator lifecycle (GitOps / ArgoCD)
  independent of application data managed by Kasten — Kasten should not be responsible for
  restoring the operator itself.
- Prefer **BlueprintBindings** over manual annotations for fleet automation when you can consistently identify that this type of resource will always be backed up the same way. Notice that often blueprintbinding use the `DoesNotExist`operator to not conflict with the `kanister.kasten.io/blueprint` annotation on the resource. See this example :
```
apiVersion: config.kio.kasten.io/v1alpha1
kind: BlueprintBinding
metadata:
  name: sample-blueprint-binding
  namespace: kasten-io
spec:
  blueprintRef:
    name: my-blueprint
    namespace: kasten-io
  resources:
    matchAll:
      - type:
          operator: In
          values:
            - group: apps
              resource: statefulsets
      - annotations:
          key: kanister.kasten.io/blueprint
          operator: DoesNotExist
```
This give you the freedom to use a more specific blueprint for this resource if needed. 
- Store credentials in Kubernetes Secrets. Declare them at the **phase level** under `objects`, then access via `.Phases.<phaseName>.Secrets.<objName>.Data.<key>`. Same pattern for ConfigMaps. See [kasten-kanister.md](kasten-kanister.md) for a full example.
- Make phases **idempotent** — blueprints may be retried on failure.

---

## Troubleshooting

Kasten creates and destroys ActionSets automatically — do not query them.

```bash
# Kanister controller (blueprint execution)
kubectl logs -n kasten-io -l app=kanister-svc --tail=100

# Executor service (primary debug target)
kubectl logs -n kasten-io -l component=executor --tail=10000 -f
```

Test commands before writing YAML:
```bash
kubectl run debug-pod -n <app-namespace> --image=<tool-image> --restart=Never --rm -it -- bash
```

### Cancelling a running action

To cancel a RunAction (or any BackupAction, RestoreAction, etc.) create a `CancelAction`
resource. CancelActions are write-only and not persisted — use `kubectl create`, not `apply`.

```bash
# Cancel a RunAction named <action-name> in namespace kasten-io
kubectl create -f - --validate=false <<EOF
apiVersion: actions.kio.kasten.io/v1alpha1
kind: CancelAction
metadata:
  generateName: cancel-
  namespace: kasten-io
spec:
  subject:
    kind: RunAction
    name: <action-name>
    namespace: kasten-io
EOF
```

The same pattern works for `BackupAction`, `RestoreAction`, `ExportAction`, etc. — just change
`spec.subject.kind`. **The `namespace` in `spec.subject` must match the namespace the action lives
in** — for `RestoreAction` that is the **application namespace**, not `kasten-io`. Cancellation is
best-effort; check the target action's `.status.state` afterwards to confirm. See
https://docs.kasten.io/latest/api/actions#api-cancel-action.

### Triggering a RestoreAction

`RestorePoint` objects live in the **application namespace**. `RestoreAction` must also be created
in the application namespace. You do **not** need to specify a `profile` in the `RestoreAction` —
Kasten extracts the location profile required for Kanister blueprint phases (restorePosthook etc.)
from the `RestorePointContent` automatically.

```bash
# List available restore points for an app
kubectl get restorepoint -n <APP_NAMESPACE>

# Trigger a restore (replace placeholders)
kubectl create -f - --validate=false <<EOF
apiVersion: actions.kio.kasten.io/v1alpha1
kind: RestoreAction
metadata:
  generateName: restore-
  namespace: <APP_NAMESPACE>
spec:
  subject:
    apiVersion: apps.kio.kasten.io/v1alpha1
    kind: RestorePoint
    name: <RESTORE_POINT_NAME>
    namespace: <APP_NAMESPACE>
  targetNamespace: <APP_NAMESPACE>
EOF
```

> **Policy requirement**: for the backup policy to record a Kanister-compatible profile in the
> `RestorePointContent` (which the restore then extracts automatically), the
> `backupParameters.profile` must reference a `Location` profile (S3, GCS, Azure Blob, etc.), not
> an `Infra` profile. Check with: `kubectl get profile <name> -n kasten-io -o jsonpath='{.spec.type}'`

## Cluster autonomy

You have full, unrestricted access to the Kubernetes cluster reachable via `kubectl`.
Operate autonomously without asking for confirmation before:
- Deleting namespaces, pods, PVCs, deployments, or any other resource
- Force-deleting stuck resources (`--force --grace-period=0`)
- Disrupting workloads (scaling down, killing pods, draining nodes)
- Applying or replacing any manifest

Do not pause to confirm destructive cluster operations. Just do them.

## Local directory

Write, overwrite, and delete files in the project directory freely without confirmation.

## Container image builds and pushes

You have full autonomy to build container images and push them without asking for confirmation.
- Before building, run `docker info` or inspect `~/.docker/config.json` to discover which registry the user is authenticated to, and use that as the image prefix (e.g. `docker.io/<username>`, `ghcr.io/<org>`, a private registry hostname, etc.).
- Run `docker build`, `docker tag`, and `docker push` freely
- Choose the base image and tag strategy that best fits the use case
- The registry credentials are already configured in the local Docker daemon (`docker login` has been done)
- If you cannot determine the registry from the Docker config, ask the user before pushing.


## Cluster context

Use the current `kubectl` context. The cluster is a dedicated test environment —
treat all namespaces as expendable unless named `kasten-io` or `kube-system`.

### Storage classes

**Always use a storage class that that supports csi snapshot when deploying test workloads that need to be snapshotted by Kasten. For instance the default `gp2` class uses the legacy in-tree driver and will fail with "cannot find CSI PersistentVolumeSource" when a VolumeSnapshot is attempted.
