# ansible-playbook-sap

[![Linter](https://github.com/thbe/ansible-playbook-sap/actions/workflows/linter.yml/badge.svg)](https://github.com/thbe/ansible-playbook-sap/actions/workflows/linter.yml)

Operational playbooks for SAP systems on RHEL: instance control, Pacemaker high
availability resource configuration (HANA and NetWeaver), and Azure/NFS disk
provisioning for SAP workloads.

This repository is normally consumed as the `playbooks/sap` git submodule of
[ansible-main](https://github.com/thbe/ansible-main), but it can also be used
standalone.

## Table of Contents

- [Requirements](#requirements)
- [Playbooks](#playbooks)
  - [Instance control](#instance-control)
  - [High availability (Pacemaker)](#high-availability-pacemaker)
  - [Disk provisioning](#disk-provisioning)
- [Variables](#variables)
- [Usage](#usage)
- [License](#license)
- [Author](#author)

## Requirements

- Ansible 2.14+ (core) with the collections:
  - `ansible.posix`
  - `community.general`
- Target hosts running RHEL for SAP with the SAP control wrapper
  `/opt/sap/bin/sap.sh` present (for the instance-control playbooks).
- For the HA playbooks: a configured Pacemaker/`pcs` cluster.
- Privilege escalation (`become`) available. All playbooks target `all`; scope
  each run with `--limit`.

## Playbooks

### Instance control

Thin wrappers around `/opt/sap/bin/sap.sh` (idempotent, `changed_when: false`).

| Playbook          | Description                     |
| ----------------- | ------------------------------- |
| `sap_start.yml`   | Start the SAP instance(s).      |
| `sap_stop.yml`    | Stop the SAP instance(s).       |
| `sap_health.yml`  | Report SAP instance status.     |

### High availability (Pacemaker)

| Playbook                                 | Description                                                                                 |
| ---------------------------------------- | ------------------------------------------------------------------------------------------- |
| `cluster/sap_ha_resource_hana.yml`       | Create HANA cluster resources: `SAPHanaTopology`, `SAPHana`, virtual IPs, ordering/colocation constraints, and optional Azure NetApp Files (ANF) filesystem resources. |
| `cluster/sap_ha_resource_netweaver.yml`  | Create NetWeaver ASCS/SCS/ERS resources, virtual IPs, `azure-lb` and `SAPInstance` resources with the required constraints. |

These playbooks are guarded so that tasks run only on the designated primary or
secondary node (`fqdn == sap_ha_primary/secondary`) and only when `sap_ha` is
enabled.

### Disk provisioning

| Playbook                              | Description                                                                                     |
| ------------------------------------- | ----------------------------------------------------------------------------------------------- |
| `disks/sap_shared_setup.yml`          | Mount shared NFS volumes (`/usr/sap/trans`, `/media/sap`, `/media/backup`, `/sapdata/<SID>`).    |
| `disks/azure/sap_hana_setup.yml`      | Provision HANA disks: striped LVM (`/hana/shared|log|data`) for local disks, or ANF NFS mounts.  |
| `disks/azure/sap_netweaver_setup.yml` | Provision NetWeaver disks (`/usr/sap`, `/sapmnt`) via LVM or ANF NFS mounts.                      |

## Variables

These playbooks are driven primarily by inventory `host_vars`/`group_vars`. Key
variables include:

- **Instance:** `sap_instance_sid`, `sap_instance_java_sid`, `sap_instance_type`
  (`ASCS`/`DB`/`Router`/`Webdispatcher`/...), `sap_instance_product`, `sap_alias`.
- **HA:** `sap_ha`, `sap_ha_primary`, `sap_ha_secondary`, `sap_ha_hana_instance_number`,
  `sap_ha_ascs/scs/aers/ers_instance_number`, `sap_ha_vip_primary/secondary`,
  `sap_ha_vcidr_primary/secondary`.
- **Storage:** `anf_storage`, `sap_anf`, `sap_anf_options`, `nfs_options`, and the
  various `sap_anf_hana_*` / `sap_anf_netweaver_*` NFS source variables.

See the inventory repositories (`ansible-inventory-*`) for concrete examples.

## Usage

```shell
# Check SAP status on the HANA hosts
ansible-playbook -i inventories/prod/hosts.yml sap_health.yml --limit hana

# Provision HANA disks on a new DB host
ansible-playbook -i inventories/prod/hosts.yml disks/azure/sap_hana_setup.yml --limit hanadb01

# Configure HANA cluster resources
ansible-playbook -i inventories/prod/hosts.yml cluster/sap_ha_resource_hana.yml --limit hana
```

## License

GPL-3.0-only

## Author

Thomas Bendler - [https://www.thbe.org/](https://www.thbe.org/)
