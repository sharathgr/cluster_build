Ansible NetApp Cluster Build

Overview:
- Playbook: netapp_cluster_build.yml
- Role: roles/netapp
- Place NetApp API credentials in group_vars/all.yml

Quick start:
1. Install collections:

```bash
ansible-galaxy collection install -r requirements.yml
Ansible NetApp Cluster Build

**Purpose**
- Automate NetApp ONTAP cluster configuration using REST API calls (target: ONTAP 9.16.1).
- Provide a reusable role and playbook to create snapshot policies, modify volumes, configure CIFS/NFS, manage export-policies, create security accounts (AD groups), manage local CIFS group membership, and configure cluster logging / EMS notifications.

**Repository layout**
- Playbook: [ansible/netapp_cluster_build.yml](ansible/netapp_cluster_build.yml#L1)
- Role tasks: [ansible/roles/netapp/tasks/main.yml](ansible/roles/netapp/tasks/main.yml#L1)
- Inventory: [ansible/inventory/hosts.ini](ansible/inventory/hosts.ini#L1)
- Example variables: [ansible/group_vars/all.yml](ansible/group_vars/all.yml#L1)
- Collections list: [ansible/requirements.yml](ansible/requirements.yml#L1)

**Supported operations (mapped from CLI-style commands to ONTAP REST endpoints)**
- Snapshot policy create
	- CLI example: `snapshot policy create -policy policy_01_i_35d ...`
	- REST endpoint: `/api/storage/snapshot-policies` (implemented in role tasks)

- Volume rename / modify
	- CLI examples: `vol rename`, `vol modify -size -percent-snapshot-space -snapshot-policy -snapshot-locking-enabled`
	- REST endpoint: `/api/storage/volumes` (GET to find by name, PATCH to update)

- CIFS configuration (advertised encryption, LM compatibility, signing, client timeout)
	- REST endpoint: `/api/protocols/cifs/services/` (GET + PATCH)

- NFS configuration (idle connection timeout, access, v4.0/v4.1 toggle)
	- REST endpoint: `/api/protocols/nfs/services/{svm.uuid}`

- Vserver (SVM) modify (language and other SVM settings)
	- REST endpoint: `/api/svm/svms/{uuid}` (role provides places to add SVM patching)

- Export policy rule modifications, renames, and deletes
	- REST endpoint: `/api/protocols/nfs/export-policies/{id}`

- Security accounts (create AD group/service mappings for `http`, `ssh`, `ontapi`)
	- REST endpoint: `/api/security/accounts`

- CIFS local groups (add/remove members)
	- REST endpoints: `/api/protocols/cifs/local-groups/{svm.uuid}/{local_cifs_group.sid}/members`

- Export-policy deletion
	- REST endpoint: `/api/protocols/nfs/export-policies/{id}` (DELETE)

- Cluster log forwarding / syslog destinations
	- REST endpoint: `/api/security/audit/destinations/{address}/{port}` (PATCH)

- EMS (event) destinations, filters and rules, notifications
	- REST endpoints: `/api/support/ems/destinations`, `/api/support/ems/filters`, `/api/support/ems/notifications`

These mappings and example payloads in the role are aligned to ONTAP 9.16.1 REST API conventions — see NetApp docs for exact request/response schemas.

**Prerequisites**
- Ansible (2.9+ recommended; newer releases are fine).
- Install required collections:

```bash
ansible-galaxy collection install -r ansible/requirements.yml
```

- Network connectivity from control host to the NetApp management endpoint(s).

**Configuration / variables**
- Edit `[ansible/group_vars/all.yml](ansible/group_vars/all.yml#L1)` and set:
	- `netapp_host` — management IP or DNS name of the cluster management endpoint
	- `netapp_username`, `netapp_password` — credentials for REST calls (consider Ansible Vault)
	- Data structures for `snapshot_policies`, `volumes`, `cifs_settings`, `nfs_settings`, `export_policies`, `security_accounts`, `cifs_group_members_add`, `cifs_group_members_remove`, `syslog_destinations`, `ems_destinations`, `ems_filters`, `ems_notifications`.

Examples and schema are provided in `ansible/group_vars/all.yml` as a starting point. The role uses lookups (GET) where necessary to translate names to UUIDs/IDs before PATCH/DELETE operations, but you should validate results for your environment.

**Running the playbook**
- Dry-run (check mode):

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/netapp_cluster_build.yml --check
```

- Actual run:

```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/netapp_cluster_build.yml
```

**Security and credentials**
- Avoid committing `netapp_password` to VCS. Use Ansible Vault to encrypt secrets:

```bash
ansible-vault encrypt ansible/group_vars/all.yml
ansible-playbook -i ansible/inventory/hosts.ini ansible/netapp_cluster_build.yml --ask-vault-pass
```

**Testing and validation**
- This role relies on Ansible `uri` to call NetApp REST endpoints; runtime validation requires access to a test cluster or a NetApp simulator. You can extend the project with `molecule` for role testing, or add CI jobs that run `ansible-lint` and `ansible-playbook --syntax-check`.

**Notes, caveats and next steps**
- API schemas and field names can change between ONTAP patch levels. The role is targeted at 9.16.1; if your cluster is another release, review the JSON payloads in [ansible/roles/netapp/tasks/main.yml](ansible/roles/netapp/tasks/main.yml#L1) and adjust field names accordingly.
- Some operations require resource UUIDs; the role attempts lookups but you may prefer to provide explicit UUIDs for deterministic runs.
- If you want, I can add:
	- Ansible Vault integration examples and a separate `secrets.yml` file
	- Molecule tests and a CI workflow
	- More precise payloads for additional ONTAP endpoints (if you provide API examples or responses)

**Support / References**
- NetApp ONTAP REST API docs (example): https://docs.netapp.com

If you'd like, I can now add Ansible Vault support and show encryption steps, or add a small CI/linting workflow. Let me know which next step you prefer.

**Ansible Tower / AWX notes**
- This project is compatible with Ansible Tower / AWX. Recommended Tower job-template setup:
	- Inventory: [ansible/inventory/hosts.ini](ansible/inventory/hosts.ini#L1) (or your Tower inventory)
	- Project: point to this repository
	- Credentials: store NetApp credentials (use machine credential type or a custom credential for API access)
	- Survey / Extra Vars: add a survey variable named `svm_name` (string) for a single SVM, or `svm_names` (list) for multiple SVMs.
		- The playbook normalizes these into `svm_list` (see `netapp_cluster_build.yml`).
	- Example survey variables (single SVM):

	```text
	svm_name: my-svm-01
	netapp_host: my-cluster-mgmt.example.com
	netapp_username: admin
	netapp_password: <use credential or prompt>
	```

	- For multiple SVMs, provide `svm_names` as a list (Tower supports multi-select survey options or accept comma-separated list via extra-vars and map to `svm_names`).

- When the Tower job runs, `svm_name` / `svm_names` are available to templates and the role uses `{{ svm_name }}` in `group_vars/all.yml` samples and `svm_list` for per-SVM iteration.

If you want, I can generate a sample Tower job template JSON for import or add a template `tower/job_template.yml` to this repo.
