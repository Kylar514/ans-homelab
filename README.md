# Ansible Homelab

Ansible automation for a self-managing homelab infrastructure.

## Overview

- **Control node:** Azir (`10.0.40.11`)
- **Remote user:** `zeal`
- **Target OS:** Ubuntu 22.04+ (jammy / noble)
- **Min Ansible:** 2.14

## Quick Start

```bash
# Run against production
ansible-playbook playbooks/site.yml --vault-password-file ~/.ansible_vault_pass

# Dry run
ansible-playbook playbooks/site.yml --check --diff --vault-password-file ~/.ansible_vault_pass
```

## Structure

```
ans-homelab/
├── playbooks/          # Ansible playbooks
│   └── site.yml        # Master playbook
├── roles/              # Ansible roles
├── inventories/        # Production, staging, dmz
└── .github/workflows/ # CI/CD
```

## Hosts

- **Azir:** Control node
- **Ryze, Zilean, Kennen:** Production VMs
- **Heimerdinger:** Staging VM

## See Also

- [AGENTS.md](./AGENTS.md) - Agent-specific documentation
