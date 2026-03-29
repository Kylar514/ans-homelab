# ans-homelab

Ansible automation for self-managing homelab infrastructure.

## Overview

- **Control node:** azir-01 (10.0.40.11)
- **Remote user:** `ansible`
- **Target OS:** Ubuntu 22.04+ (jammy / noble)
- **Min Ansible:** 2.14
- **CI/CD:** Self-hosted GitHub Actions runner
- **User:** `ansible` (passwordless sudo, SSH key-based access)

> NOTE: Hosts are provisioned via Terraform using default cloud-init. Ensure SSH
> access and initial user setup before running bootstrap.

### What This Project Demonstrates

- Infrastructure as Code (Terraform + Ansible)
- Configuration management with idempotent roles
- Inventory-driven architecture and variable precedence design
- CI/CD automation using self-hosted GitHub Actions runners
- Kubernetes (K3s) cluster provisioning and storage integration (NFS)
- Secure secret management using Ansible Vault

### Conventions

- **Variables**: All variables are stored in `inventory/..`
  - Variables prefixed with `all_` and `group_` are **merged additively** (e.g.,
    lists are combined and deduplicated)
  - All other variables follow standard precedence: `group_vars` > `all` >
    `role_defaults`
- **Roles**: Logic is kept to roles
- **Inventory**: Naming convention is as follows;

| Group             | Hosts                        | Purpose                     |
| ----------------- | ---------------------------- | --------------------------- |
| `all`             | All 5 hosts                  | Global settings             |
| `k3s`             | kaiju-01, corps-01, corps-02 | Kubernetes cluster nodes    |
| `ansible_control` | azir-01                      | Control node with GH runner |
| `storage`         | nasus                        | NFS server                  |
| `corps`           | corps-01, corps-02           | K3s Worker nodes            |
| `kaijus`          | kaiju-01                     | K3s Control Plane           |

### CI/CD

Pushes to `master` trigger automatic deployment:

1. GitHub Actions runner (hosted on the control node) detects changes
2. Executes `site.yml` against the full inventory
3. Applies desired state with no manual intervention

- This repository is the **single source of truth** for infrastructure state
- Starting process requires a bootstrap ran, which configures the GH Runner and
  clones the repo.

### TODOs

- Add monitoring (Grafana) for configuration drift detection
- Implement Molecule tests for role validation
- Integrate ansible-lint and Molecule into CI pipeline

## Quick Start

```bash
# Bootstrap (initial setup - run once)
ansible-playbook playbooks/bootstrap.yml --ask-vault-pass

# Regular maintenance (via CI or manual)
ansible-playbook playbooks/site.yml

# Dry run
ansible-playbook playbooks/site.yml --check --diff
```

## Structure

```
ans-homelab/
├── playbooks/
│   ├── bootstrap.yml     # Initial setup (users, SSH, GH runner)
│   └── site.yml          # Ongoing maintenance
├── roles/                # Ansible roles
├── inventories/
│   └── production/
│       ├── inventory.yml          # Host definitions
│       └── group_vars/            # Group-specific variables
│           ├── all/               # All hosts
│           ├── k3s/               # Kubernetes nodes
│           ├── ansible_control/   # Control node
│           └── ...
└── .github/workflows/    # CI/CD automation
```

## Related Projects

- [terraform-homelab](https://github.com/kylar514/tf-homelab) - Infrastructure
  provisioning
- [k3s-homelab](https://github.com/kylar514/k3s-homelab) - Kubernetes deployment
  (coming soon)
- [Blog Post](https://kyle.kliamovich.com) - Full write-up on the complete
  (coming soon) homelab setup

## License

MIT-0
