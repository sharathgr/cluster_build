# Ansible NetApp Cluster Build

## Purpose
Automate NetApp ONTAP cluster configuration using REST API calls (target: ONTAP 9.16.1). This project provides a reusable Ansible role and playbook to:
- Create snapshot policies with custom schedules and retention
- Rename and modify volumes (size, snapshot space, policies, locking)
- Configure CIFS services (encryption, LM compatibility, signing, timeouts)
- Configure NFS services (idle timeouts, access, protocol versions)
- Manage export policies (create, modify, rename, delete rules)
- Create security accounts (AD group/service mappings for HTTP, SSH, ONTAPI)
- Manage CIFS local group membership (add/remove members)
- Configure cluster log forwarding to syslog destinations
- Configure EMS (event management) destinations, filters, and notifications

## Repository layout
```
cluster_build/
├── playbooks/
│   └── netapp_cluster_build.yml       # Main playbook
├── roles/
│   └── netapp/
│       ├── tasks/
│       │   └── main.yml               # Role tasks (REST API calls)
│       └── README.md
├── inventory/
│   └── hosts.ini                      # Ansible inventory
├── group_vars/
│   └── all.yml                        # Example variables (Tower-friendly)
├── requirements.yml                   # Ansible collections
├── .gitignore                         # Git ignore rules
└── README.md                          # This file
```

## Supported operations (mapped to ONTAP REST endpoints)

### Snapshot Policy Management
- **Operation**: Create snapshot policies with multiple schedules
- **CLI example**: `snapshot policy create -policy policy_01_i_35d -enabled true -schedule1 daily -count1 35 ...`
- **REST endpoint**: `POST /api/storage/snapshot-policies`
- **Payload fields**: `name`, `is_enabled`, `comment`, `rules[]`

### Volume Operations
- **Operation**: Find, rename, and modify volumes
- **CLI examples**: 
  - `vol rename -vserver <vserver_name> -volume <vserver_name>_root -newname vsroot__<vserver_short_name>`
  - `vol modify -vserver <vserver_name> -volume <vserver_name>_root -size 2GB -percent-snapshot-space 50 -snapshot-policy policy_01_i_35d -snapshot-locking-enabled true`
- **REST endpoints**: 
  - `GET /api/storage/volumes?name=<vol>&svm.name=<vserver>` (find by name)
  - `PATCH /api/storage/volumes/<uuid>` (modify)
- **Payload fields**: `name`, `size`, `percent_snapshot_space`, `snapshot_policy.name`, `is_snapshot_locking_enabled`

### CIFS Services Configuration
- **Operations**: Configure advertised encryption types, LM compatibility, signing, client session timeouts
- **CLI example**: `cifs security modify -vserver <vserver_name> -advertised-enc-types aes-256,aes-128 -lm-compatibility-level ntlmv2-krb -is-signing-required false`
- **REST endpoints**:
  - `GET /api/protocols/cifs/services?svm.name=<vserver>` (find service UUID)
  - `PATCH /api/protocols/cifs/services/<uuid>` (modify)
- **Payload fields**: `advertised_encryption_types`, `lm_compatibility_level`, `is_signing_required`, `client_session_timeout`

### NFS Services Configuration
- **Operations**: Configure idle timeouts, access, NFSv4.0/v4.1 toggle
- **CLI example**: `nfs modify -vserver <vserver_name> -idle-connection-timeout 300 -access true -v4.0 disabled -v4.1 disabled`
- **REST endpoints**:
  - `GET /api/protocols/nfs/services?svm.name=<vserver>` (find service UUID)
  - `PATCH /api/protocols/nfs/services/<uuid>` (modify)
- **Payload fields**: `idle_connection_timeout`, `access`, `v4_0`, `v4_1`

### Export Policy Management
- **Operations**: Find, modify, rename export policies and rules
- **CLI examples**: 
  - `export-policy rule modify -vserver <vserver_name> -policyname default -ruleindex 1 -protocol any -rorule never -rwrule never -superuser none`
  - `export-policy rename -vserver <vserver_name> -policyname fsx-root-volume-policy -newpolicyname /`
  - `export-policy delete -vserver * -policyname fsx-root-volume-policy`
- **REST endpoints**:
  - `GET /api/protocols/nfs/export-policies?svm.name=<vserver>&name=<policyname>` (find)
  - `PATCH /api/protocols/nfs/export-policies/<id>` (modify rules/rename)
  - `DELETE /api/protocols/nfs/export-policies/<id>` (delete)
- **Payload fields**: `name`, `rules[]`

### Security Accounts (Active Directory Groups)
- **Operations**: Create domain account mappings for HTTP, SSH, ONTAPI applications
- **CLI examples**: 
  - `security login create -user-or-group-name abc\ad_group1 -application http -authentication-method domain -role fsxadmin-readonly`
  - `security login create -user-or-group-name abc\ad_group1 -application ssh -authentication-method domain -role fsxadmin-readonly`
  - `security login create -user-or-group-name abc\ad_group1 -application ontapi -authentication-method domain -role fsxadmin-readonly`
- **REST endpoint**: `POST /api/security/accounts`
- **Payload fields**: `user_or_group_name`, `application`, `authentication_method`, `role`

### CIFS Local Group Membership
- **Operations**: Add/remove members from BUILTIN groups (Administrators, Backup Operators)
- **CLI examples**:
  - `cifs users-and-groups local-group add-members -vserver <vserver_name> -group-name "BUILTIN\Administrators" -member-names "abc\tom"`
  - `cifs users-and-groups local-group remove-members -vserver <vserver_name> -group-name "BUILTIN\Administrators" -member-names "abc\domain Admins"`
- **REST endpoints**:
  - `POST /api/protocols/cifs/local-groups/<svm.uuid>/<group.sid>/members` (add)
  - `DELETE /api/protocols/cifs/local-groups/<svm.uuid>/<group.sid>/members/<name>` (remove)
- **Payload fields**: `name` (member name)

### Cluster Log Forwarding (Syslog)
- **Operations**: Configure syslog destinations for audit and event logs
- **CLI examples**:
  - `cluster log-forwarding create -destination 10.10.10.10 -port 7501 -protocol udp-unencrypted -ipspace Default`
  - `cluster log-forwarding create -destination 11.11.11.11 -port 7927 -protocol tcp-unencrypted -ipspace Default`
- **REST endpoint**: `PATCH /api/security/audit/destinations/<address>/<port>`
- **Payload fields**: `protocol`, `ipspace`

### EMS (Event Management System) Configuration
- **Operations**: Create EMS destinations, filters, and notification rules
- **CLI examples**:
  - `event notification destination create -name Cribl -syslog 10.10.10.10 -syslog-port 7501 -syslog-transport udp-unencrypted`
  - `event notification destination create -name Splunk -syslog 11.11.11.11 -syslog-port 7927 -syslog-transport tcp-unencrypted`
  - `event filter create -filter-name all-events`
  - `event filter rule add -filter-name all-events -type include -message-name * -severity EMERGENCY,ERROR,ALERT,NOTICE,INFORMATIONAL,DEBUG -position 1 -snmp-trap-type *`
  - `event notification create -filter-name all-events -destinations Cribl,Splunk`
- **REST endpoints**:
  - `POST /api/support/ems/destinations` (create destination)
  - `POST /api/support/ems/filters` (create filter)
  - `PATCH /api/support/ems/filters/<name>` (add rules)
  - `POST /api/support/ems/notifications` (create notification)
- **Payload fields**: `name`, `syslog`, `syslog_port`, `syslog_transport`, `rules[]`, `filter_name`, `destinations[]`

## Prerequisites
- **Ansible**: 2.9+ (newer versions supported)
- **Collections**: NetApp ONTAP and Community.General (installed via `requirements.yml`)
- **Network**: Connectivity from control host to NetApp cluster management IP/hostname
- **Credentials**: NetApp cluster admin or equivalent REST API user account

## Installation & Setup

### 1. Install Ansible collections
```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure inventory
Edit `inventory/hosts.ini` and replace placeholders:
```ini
[netapp]
netapp1 ansible_host=your-netapp-mgmt-ip-or-hostname.example.com

[netapp:vars]
ansible_connection=local
```

### 3. Configure variables
Edit `group_vars/all.yml` and set:
- `netapp_host`: NetApp cluster management IP/hostname
- `netapp_username`: REST API user
- `netapp_password`: REST API password (use Ansible Vault - see Security section)
- Data structures for policies, volumes, CIFS/NFS settings, export policies, security accounts, group membership, syslog, and EMS

Example snippet:
```yaml
netapp_host: "192.168.1.100"
netapp_username: "admin"
netapp_password: "encrypted_via_vault"  # Use ansible-vault encrypt group_vars/all.yml

snapshot_policies:
  - name: policy_01_i_35d
    enabled: true
    comment: "Standard Snapshot Policy - 35 Daily / 4 Weekly / 1 Monthly"
    rules:
      - schedule: daily
        count: 35
        snapmirror_label: daily
        retention_period: "35days"
      - schedule: weekly
        count: 4
        snapmirror_label: weekly
      - schedule: monthly
        count: 1
        snapmirror_label: monthly
```

## Running the Playbook

### Dry-run (check mode - no changes applied)
```bash
ansible-playbook -i inventory/hosts.ini playbooks/netapp_cluster_build.yml --check
```

### Actual run (applies configuration)
```bash
ansible-playbook -i inventory/hosts.ini playbooks/netapp_cluster_build.yml
```

### Run with Ansible Vault (encrypted passwords)
```bash
ansible-playbook -i inventory/hosts.ini playbooks/netapp_cluster_build.yml --ask-vault-pass
```

### Run with extra variables (Tower/AWX compatible)
```bash
ansible-playbook -i inventory/hosts.ini playbooks/netapp_cluster_build.yml -e "svm_name=my-svm-01"
```

## Security & Credentials

### DO NOT commit plaintext passwords
1. **Encrypt sensitive data**:
```bash
ansible-vault encrypt group_vars/all.yml
```

2. **Create a `.vault_password_file`** (local only, add to `.gitignore`):
```bash
echo "your-vault-password" > .vault_password_file
chmod 600 .vault_password_file
```

3. **Run with vault password**:
```bash
ansible-playbook -i inventory/hosts.ini playbooks/netapp_cluster_build.yml --vault-password-file .vault_password_file
```

### In Ansible Tower / AWX
- Use **Machine Credential** or **Custom Credential** types to store NetApp credentials
- Do NOT embed passwords in survey variables
- Pass credentials via Tower credential assignment, not in extra-vars

## Ansible Tower / AWX Integration

This project is fully compatible with Ansible Tower / AWX. Recommended setup:

### Job Template Configuration
- **Inventory**: Use `inventory/hosts.ini` or import into Tower
- **Project**: Point to this repository
- **Playbook**: `playbooks/netapp_cluster_build.yml`
- **Credentials**: Create or select Machine Credential with NetApp admin account
- **Options**: 
  - Enable `Privilege Escalation` if needed
  - Enable `Verbose` logging for debugging

### Tower Survey (Extra Variables)
Add a survey with these fields to make runs parameterized:

```
svm_name:
  Type: Text
  Label: "SVM Name (e.g., vs1, my-svm-01)"
  Default: vs1
  Required: Yes

netapp_host:
  Type: Text
  Label: "NetApp Cluster Management IP or Hostname"
  Default: netapp.example.com
  Required: Yes

netapp_username:
  Type: Text
  Label: "REST API Username"
  Default: admin
  Required: Yes

netapp_password:
  Type: Password
  Label: "REST API Password"
  Required: Yes
```

### Tower Pre-task
The playbook has a pre_task that normalizes SVM input:
- If `svm_name` is provided: uses single SVM
- If `svm_names` (list) is provided: iterates over multiple SVMs
- Passes `svm_list` to the role for per-SVM configuration

### Example Tower Job Run
```
Extra Variables (from survey):
svm_name: prod-svm-01
netapp_host: 192.168.1.100
netapp_username: admin
netapp_password: <credential-provided>
```

The playbook will then apply all configured policies, volumes, CIFS/NFS settings, and group membership to `prod-svm-01`.

## Testing & Validation

### Syntax check
```bash
ansible-playbook -i inventory/hosts.ini playbooks/netapp_cluster_build.yml --syntax-check
```

### Lint with ansible-lint
```bash
ansible-lint playbooks/netapp_cluster_build.yml roles/netapp/tasks/main.yml
```

### Dry-run with verbose output
```bash
ansible-playbook -i inventory/hosts.ini playbooks/netapp_cluster_build.yml --check -vvv
```

### Test on simulator or lab cluster
- Use NetApp simulator or lab cluster to validate before production
- Review Ansible output for any REST API errors or failed tasks
- Verify ONTAP configuration matches expectations via CLI or System Manager

## Troubleshooting

### REST API Connection Issues
- Verify `netapp_host` is reachable from control host: `ping <netapp_host>`
- Check REST API user credentials (admin or equivalent role required)
- Ensure `validate_certs: no` is appropriate for your environment (consider using proper certificates in production)

### Task Failures
- Check Ansible verbose output: `-vvv` flag
- Review role tasks in `roles/netapp/tasks/main.yml` for the exact REST endpoint and payload
- Compare payload structure against NetApp ONTAP 9.16.1 REST API documentation

### SVM / Volume Not Found
- The role attempts to look up resources by name via GET queries
- If lookups fail, provide explicit UUIDs in the configuration
- Verify SVM and volume names match those on the cluster

## Notes & Limitations

### ONTAP Version
- **Target**: ONTAP 9.16.1
- If running on a different ONTAP version, review and adjust JSON payloads in `roles/netapp/tasks/main.yml`
- API field names and structures may differ between versions

### Resource Lookups
- The role uses REST GET queries to find volumes, policies, and services by name
- For deterministic, repeatable runs, provide explicit UUIDs instead of names
- Lookups may fail if resources don't exist; the `ignore_errors: yes` in tasks prevents playbook failure

### API Limitations
- Some operations require cluster-level or SVM-level permissions
- Ensure your REST API user account has appropriate roles assigned
- Check ONTAP documentation for required privileges per operation

## Advanced Usage

### Extend the role
Add more operations by extending `roles/netapp/tasks/main.yml` with additional `uri` module tasks following the pattern of existing tasks.

### Use with CI/CD
Add a GitHub Actions or GitLab CI workflow to:
- Run `ansible-lint` on commits
- Validate playbook syntax
- Test against a simulator in CI environment

### Integrate with Terraform or Ansible Automation Platform
- Use this role as part of larger IaC pipelines
- Call the playbook from parent workflows or Terraform modules

## Support & References

- **NetApp ONTAP REST API Docs**: https://docs.netapp.com/us-en/ontap-restapi-9161/
- **Ansible Documentation**: https://docs.ansible.com/
- **Ansible NetApp Collections**: https://github.com/ansible-collections/netapp.ontap

## License & Contributing

This project is provided as-is for NetApp ONTAP cluster automation using REST APIs (ONTAP 9.16.1 target).

---

**Project Status**: ✅ Ready for Tower/AWX integration, Ansible CLI, and CI/CD workflows.
