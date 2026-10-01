# AAP backup and recovery sample (OADP on OpenShift)

This project is a collection of simplified Ansible playbooks, and OpenShift custom resources (as Jinja2 templates), that walk through a full sample of backup and recovery of Ansible Automation Platform (AAP) on OpenShift, using the OpenShift API for Data Protection (OADP).

Backup storage is an Azure Blob container. Kubernetes objects and persistent volume data both land in that container.

## What the sample covers

OADP stores two different things:

- Cluster objects (AAP custom resources, Secrets, ConfigMaps, Routes, roles) are uploaded to the Blob container.
- Persistent volume data is copied into the same container by the OADP node agent (Kopia file-system backup). CSI disk snapshots stay in Azure as disk snapshots and are outside this sample.

For an in-cluster database, the data that matters is the PostgreSQL volume (controller, hub, and gateway state) and Automation Hub artifact storage. Execution pods, completed jobs, and events are excluded.

AAP may also use an external PostgreSQL database. This sample includes that layout: OADP backs up the in-cluster objects that connect to it (custom resources, Secrets, and hub artifact volumes). Backup and recovery of the external database itself are not part of this sample. Coordinating an AAP restore with a separate database restore — matching point in time, contents, and connection endpoints — is an operational concern and a risk. An AAP restore applied against a database that is ahead of, behind, or unrelated to that backup leaves the platform inconsistent.

Back up and restore at the same AAP version. Use AAP’s own upgrade path to move versions. The AAP operator is assumed to be installed already on every cluster that restores; the sample restores the application, not Operator Lifecycle Manager state.

Azure credentials and storage settings come from Ansible variables. Templates render the Secret and the storage location. The repository does not contain client secrets.

## Inventory

Copy [`inventory-DIST`](inventory-DIST) to `inventory` and fill in the OpenShift and Azure values. `inventory` and `d.r.c-notes` are gitignored.

`inventory` has two groups, both using a local connection:

- `openshift` (`cluster`) — API URL, console URL, kubeadmin username and password, a bearer token, and `ocp_validate_certs`. This cluster’s API certificate is not in the local trust store, so the filled-in inventory sets that to `false`. The template leaves it `true`.
- `azure` (`portal`) — portal URL, tenant, service principal client id and secret, and subscription id.

Storage account and resource group are not in this inventory yet. Later OADP scenarios will add them.

Check cluster authn and Azure portal reachability, plus service-principal auth, with:

```
ansible-playbook playbooks/smoke.yml
```

The smoke test prints the OpenShift identity and the Azure subscription name and state. It does not log tokens or client secrets. Reaching `https://portal.azure.com` proves the portal is reachable. Portal sign-in is Red Hat SSO; the service principal is checked against Azure Resource Manager, not the portal UI.

## Scenarios

Each scenario is a playbook that renders one or more custom resources from a Jinja2 template, applies them, and verifies the outcome. “Recovered” means the controller API accepts a login and runs a job, and hub artifacts are still present.

### 1. Stand up OADP against one Blob container

Creates the OADP namespace, an Azure credential Secret, and a `DataProtectionApplication`. Variables are the storage account, container, resource group, and subscription. The play waits until the `BackupStorageLocation` phase is `Available`.

This is the prerequisite for every other scenario.

### 2. Back up a healthy AAP namespace

An on-demand `Backup` includes the AAP namespace only. File-system backup is the default for volumes. Transient pods (automation jobs, receptor workers, completed pods) are excluded.

When PostgreSQL runs in the cluster, a backup hook checkpoints it before the volume copy, so the copy is more than crash-consistent. An external database is left to its own backup process; this playbook does not checkpoint or copy it.

Success: the `Backup` phase is `Completed`, Kubernetes objects and the Postgres and hub PVCs are in the container, and AAP keeps serving traffic.

### 3. Schedule the backup

The same backup spec becomes a `Schedule` (for example every six hours) with a TTL of a few days. The play lists backups and shows retention deleting an expired one.

Same design as scenario 2, with a schedule wrapped around it.

### 4. Lose the app and restore it in place

On the same cluster, delete the AAP custom resources and their PVCs, then apply a `Restore` that points at the backup from scenario 2. The play scales the AAP operator down before the restore and back up after it, so the operator does not recreate pods while Velero is restoring volumes.

Success: the controller route answers, an admin login works, a project syncs, a job template runs, and hub content on the artifact PVC is present. With an external database, that success also depends on the database having been restored to the same point by its own process.

### 5. Restore beside the original

Same backup, with a namespace map (`aap` to `aap-restored`). Production keeps running while the backup is proved.

Route hostnames are parameterized for the cluster apps domain so they do not collide with the live instance. Restored Secrets carry the original Postgres passwords, so the side-by-side stack logs in with those credentials.

Success: two AAP instances on one cluster; the restored one is reachable on a different route and has the same job history as the source.

### 6. Restore onto another cluster

A second cluster runs scenario 1 against the same container, then a `Restore` with:

- a namespace map when the target namespace differs
- a storage-class map when the target disk class differs
- new Route hosts for the target cluster’s apps domain

Cluster-scoped objects the operator created (ClusterRoles, SCCs) already exist on the target because the operator install created them.

Success: AAP comes up on the second cluster with its hub artifacts, and jobs run there. An in-cluster database is part of the restore. An external database must already be available at the matching point in time, through its own recovery process.

### 7. Restore only what broke

Two narrow restores from the same backup:

- Volumes only (in-cluster database): delete the Postgres PVC, restore that claim, and let the operator recreate the database pod. An external database has no PVC in this backup; recovering it stays outside the sample.
- Objects only: delete a Secret or the `AutomationController` custom resource, and restore objects without touching healthy volumes.

These use `includedResources` and `restorePVs` on the `Restore` template.

## Playbooks and templates

| Playbook | Templates | Proves |
| --- | --- | --- |
| Install OADP | Secret, `DataProtectionApplication` | Blob location is `Available` |
| Backup now | `Backup` | Objects and PVCs land in the container |
| Schedule | `Schedule` | Retention works |
| Restore in place | `Restore` | AAP works after the namespace is destroyed |
| Restore alongside | `Restore` with a namespace map | Rehearsal leaves production running |
| Restore elsewhere | `Restore` with namespace and storage-class maps | Recovery onto a second cluster |
| Restore a slice | `Restore` with resource or PVC filters | Partial recovery |
