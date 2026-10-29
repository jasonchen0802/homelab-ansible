# Common Role

## Features

- Install commonly used system packages.
- Configure basic system settings, such as timezone.
- Harden SSH authentication by disabling password authentication and preventing root password login.

## Requirements

- **Ansible Version**: `2.18+` (Tested on `2.21.5`)

- Collection requirement:

```bash
ansible-galaxy collection install community.general
```

## Role Variables

### Common essential packages

| Variable | Type | Description | Example |
| :--- | :--- | :--- | :--- |
| common_essential_packages | list | Packages you want to install | \- git <br> \- zsh |

## Dependencies

This role has no external role dependencies.

## Example Playbook

```yaml
---
- name: Initialize linux VMs
  hosts: all:!localhost
  vars:
    ansible_connection: ssh
    ansible_python_interpreter: /usr/bin/python3
  gather_facts: true
  roles:
    - role: common
```

## Maintenance CLI Commands

```bash
# Update packages
ansible-playbook site.yml -t update -vvvvv
```
