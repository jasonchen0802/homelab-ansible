# Proxmox VM Lifecycle Management Role

## Features
-----------

- **Full Lifecycle Management**: Support create, start, stop, and remove operations via a single tag.

- **Dynamic Disk Resizing**: Automatically inspects the imported cloud image size and dynamically expands it to the target storage capacity without duplication or shrink errors.

- **Accidental Deletion Prevention**: Built-in safety valve to block unauthorized VM purge commands.

## Requirements
---------------

- **Ansible Version**: `2.17+` (Tested on `2.21.1`)

- Collection requirement:

```bash
ansible-galaxy collection install community.proxmox

```
- Python library:

  - proxmoxer >= 2.3

  - requests

```bash
# pipx
pipx inject YOUR_ANSIBLE_VENV_NAME proxmoxer requests
```

## Role Variables
-----------------

### Proxmox API Token
| Variable | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| `proxmox_api_host` | string | PVE host IP | "192.168.1.3" |
| `proxmox_api_user` | string | PVE user | "root@pam" |
| `proxmox_api_token_id` | string | PVE API token ID | "your_token_id" |
| `proxmox_api_token_secret` | string | PVE API token secret | "your_token_secret" |
| `proxmox_validate_certs` | boolean | PVE API connection with TLS certificates | false |

```yaml
# inventory/group_vars/all/proxmox.yml
---
proxmox_api_host: "192.168.1.3"
proxmox_api_user: "root@pam"
proxmox_api_token_id: "your_token_id"
proxmox_api_token_secret: "your_token_secret"
proxmox_validate_certs: false
```

#### Proxmox API Token and user permission

- Datastore.AllocateSpace
- SDN.Use
- VM.Allocate
- VM.Audit
- VM.Config.CPU
- VM.Config.Cloudinit
- VM.Config.Disk
- VM.Config.HWType
- VM.Config.Memory
- VM.Config.Network
- VM.Config.Options
- VM.PowerMgmt

### VM Host

1. Unique ID

| Variable | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| `vm_id` | int | Unique Proxmox VM ID | 100 |
| `vm_hostname` | string | Hostname for the VM | vm-jarvis |

2. Hardware Configurations (`provision_specs.hardware.*`)

| Property | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| ├── `.cores` | int | Total CPU cores allocated | 2 |
| ├── `.memory` | int | Memory size in MB | 8192 |
| └── `.disk_size` | int | Target disk size (Safe from shrink error) | 32 |

3. Cloud-Init Integration (`provision_specs.cloudinit.*`)

| Property | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| ├── `.user` | string | Linux user | jarvis |
| ├── `.sshkeys` | string | For SSH login | "{{ lookup('file', '~/.ssh/my_ssh_key.pub') }}" |
| ├── `.ip` | string | Static IP with CIDR for cloud-init | "192.168.1.100/24" |
| └── `.gw` | string | Network gateway for cloud-init | "192.168.1.1" |

```yaml
# inventory/host_vars/vm_name.yml
---
vm_hostname: vm-jarvis
vm_id: 1000

provision_specs:
  hardware:
    cores: 2
    memory: 8192
    disk_size: 32

  cloudinit:
    user: jarvis
    sshkeys: "{{ lookup('file', '~/.ssh/my_ssh_key.pub') }}"
    ip: "192.168.1.100/24"
    gw: "192.168.1.1"
```

### VM Group
------------

- Storage Defaults (`storage_defaults.*`)

| Property | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| ├── `.efidisk.pool` | string | EFI disk pool in PVE | local-zfs |
| ├── `.cloudinit.volume_id` | string | cloud-init disk pool in PVE | "local-zfs:cloudinit" |
| ├── `.image.disk` | string | VM OS hard disk volume | "scsi0" |
| └── `.image.pool` | string | Target storage pool in PVE | "service" |

```yaml
# roles/proxmox-vm/default/main.yml
---
storage_defaults:
  efidisk:
    pool: local-zfs
    format: raw
  cloudinit:
    volume_id: "local-zfs:cloudinit"
  image:
    disk: scsi0
    format: raw
    pool: service
```

## Dependencies
---------------

This role has no external role dependencies. It runs completely via `connection: local` using Proxmox API Tokens.

## Example Playbook
-------------------

```yaml
---
- name: Provision Proxmox VMs
  hosts: all:!localhost
  connection: local
  gather_facts: false

  vars:
    proxmox_api: &proxmox_api
      api_host: "{{ proxmox_api_host }}"
      api_user: "{{ proxmox_api_user }}"
      api_token_id: "{{ proxmox_api_token_id }}"
      api_token_secret: "{{ proxmox_api_token_secret }}"
      validate_certs: "{{ proxmox_validate_certs }}"

  module_defaults:
    group/community.proxmox.proxmox: *proxmox_api

  roles:
    - role: proxmox-vm
```

## Maintenance CLI Commands
---------------------------

```bash
# Create VMs defined in inventory/hosts.yml and inventory/host_vars/vm_name.yml
ansible-playbook site.yml -t create_vm -vvv

# Start all VMs
ansible-playbook site.yml -t start_vm -vvv

# Stop all VMs
ansible-playbook site.yml -t stop_vm -vvv

# Remove VM
# (WARNING): By default, this will delete all VMs.
# To delete a specific VM, limit the execution using "-l your_vm_hostname"
ansible-playbook site.yml -e "purge_confirmation=true" -vvv
```
