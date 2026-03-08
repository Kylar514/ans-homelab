# AGENTS.md — Ansible Homelab Repository Guide

This file provides context for agentic coding agents working in this repository.

---

## Repository Overview

Ansible automation for a self-managing homelab infrastructure powered by self-hosted
GitHub Actions runners on Azir (the Ansible control node).

- **Control node:** Azir (`10.0.40.11`) — manages all other hosts
- **Remote user:** `zeal` (has sudo on all hosts)
- **SSH key:** `~/.ssh/ansible` (ed25519, passphrase-protected)
- **Target OS:** Ubuntu jammy / noble
- **Hostnames:** League of Legends champion names (azir, ryze, zilean, kennen, heimerdinger)
- **Min Ansible version:** 2.14

---

## CI/CD Workflows

All workflows run on self-hosted GitHub Actions runners on Azir.

| Workflow | Trigger | Action |
|----------|---------|--------|
| **Test CI/CD** | Push to `test` branch | SSH into heimerdinger (staging), pull `test` branch, run `site.yml` with staging inventory |
| **Ansible Test** | Pull request to `master` / push to `test` | Syntax check + dry run (`--check --diff`) against production inventory |
| **Prod CI/CD** | Push to `master` branch | SSH into prod hosts (azir, ryze, zilean, kennen), pull `master`, run `site.yml` |
| **Nightly** | Schedule (daily 02:00 UTC) | Run against test → if success, run against prod → if fail, trigger notification (placeholder) |

### Workflow Behavior

- **Test CI/CD**: Targets only staging host (heimerdinger), uses `inventories/staging/inventory.yml`
- **Prod CI/CD**: Targets all production hosts, uses `inventories/production/inventory.yml`
- **Ansible Test**: Runs `ansible-playbook --syntax-check` + `ansible-playbook --check --diff` against production inventory. Does NOT execute any changes, only validates syntax and shows pending changes
- **Nightly**: First runs staging playbook, waits for result. On success, runs production playbook. On failure, exits without running prod (notification hook commented out, ready to enable)

---

## Commands

### Local Development

```bash
# Full converge — production (default inventory in ansible.cfg)
ansible-playbook playbooks/site.yml --vault-password-file ~/.ansible_vault_pass

# Full converge — staging
ansible-playbook playbooks/site.yml -i inventories/staging \
  --vault-password-file ~/.ansible_vault_pass

# Bootstrap Azir as control node (run once from desktop after Terraform)
ansible-playbook playbooks/ansible_setup.yml --private-key ~/.ssh/homelab \
  --vault-password-file ~/.ansible_vault_pass

# Individual playbooks
ansible-playbook playbooks/base.yml
ansible-playbook playbooks/ufw.yml
ansible-playbook playbooks/docker.yml
ansible-playbook playbooks/vip_failover.yml

# Dry run / check mode
ansible-playbook playbooks/site.yml --check --diff \
  --vault-password-file ~/.ansible_vault_pass

# Syntax check
ansible-playbook playbooks/site.yml --syntax-check

# Lint (requires ansible-lint)
ansible-lint playbooks/site.yml
ansible-lint roles/

# Ad-hoc connectivity check
ansible all -m ping
ansible docker_hosts -m ping

# Limit to specific host or group
ansible-playbook playbooks/site.yml --limit ryze \
  --vault-password-file ~/.ansible_vault_pass

# Vault operations
ansible-vault encrypt inventories/production/group_vars/ansible_control/vault.yml
ansible-vault edit inventories/production/group_vars/ansible_control/vault.yml
ansible-vault view inventories/production/group_vars/ansible_control/vault.yml
```

---

## Repository Structure

```
ans-homelab/
  .github/
    workflows/                 # GitHub Actions workflows (future)
  ansible.cfg                 # private_key_file, inventory, vault_password_file
  AGENTS.md                   # This file
  playbooks/
    site.yml                  # Master converge — imports all playbooks in order
    base.yml                  # SSH keys, packages, upgrades — targets all hosts
    ansible_setup.yml         # Azir control node setup
    ufw.yml                   # Firewall rules — SSH always unconditionally allowed
    docker.yml                # Docker CE + dockersvc user/group — docker_hosts only
    vip_failover.yml          # Keepalived VIP failover — traefik_hosts only
    maintain.yml              # Self-maintenance (keychain reload, etc.)
  roles/
    ssh_access/               # authorized_keys managed via Jinja2 template
    sys_packages/             # Base packages + ansible_control conditional packages
    unattended_upgrades/      # Automatic security updates
    ufw/                      # UFW firewall — SSH rule is unconditional task
    docker/                   # Docker CE install, dockersvc group/user
    vip_failover/             # Keepalived config template + restart handler
    clone_repos/              # Generic git clone/update role
    secrets/                  # Write vault secrets to disk (any host)
    keychain/                 # Configure keychain + SSH key loading on login
    timezone/                 # Set system timezone
    actions_runner/           # [IN PROGRESS] Self-hosted GitHub Actions runner
  inventories/
    production/               # azir, ryze, zilean, kennen
    staging/                  # heimerdinger
    dmz/                      # Future use
```

---

## Code Style Guidelines

### License Headers
- Every YAML and shell file begins with: `# SPDX-License-Identifier: MIT-0`
  (space after `#` — some older scaffolded files omit the space, fix if editing)
- Followed by `---` on the next line
- Followed by a file-type comment: `# tasks file for <role>`, `# defaults file for <role>`

### Variable Naming
- `snake_case` throughout — no camelCase ever
- Role-scoped variables prefixed with role name: `docker_data_dir`, `ufw_rules`, `ssh_users`
- Vault variables prefixed with `vault_`: `vault_ansible_vault_pass`, `vault_vip_auth_pass`
- Defaults define empty scaffolds (e.g. `repos: []`) — actual values live in `group_vars`

### YAML Conventions
- `loop:` always — never `with_items:`
- `become: true` at play level — not `become: yes`
- `import_playbook:` in site.yml — never `include_playbook:`
- Octal modes always quoted strings: `mode: '0600'`, `mode: '0755'`
- Single quotes for simple variable references: `'{{ varname }}'`
- Double quotes for Jinja2 expressions: `"{{ item.field | filter }}"`
- `changed_when: false` on all `command:` tasks that only gather facts
- `no_log: true` on all tasks that handle secrets or key material

### `when:` Clauses
- Group membership: `when: "'group_name' in group_names"`
- Variable length guards: `when: some_list | length > 0`
- Always at task level — never inside module parameters
- New generic roles use `when` guards so they are safe to include in any playbook

### Handlers
- Handler `name:` must exactly match the `notify:` string
- Verb Noun title case: `Restart keepalived`
- Use `systemd:` module with `state: restarted`

### Templates
- First line of every template: `# Managed by Ansible - do not edit manually`
- `src:` uses filename only — Ansible resolves `templates/` directory automatically

### Playbook Structure
- Order: `name:` → `hosts:` → `become: true` → `roles:`
- Usage comment block at top of each playbook file
- `serial: 1` for HA rolling updates (traefik_hosts)

### SSH Keys
- ed25519 only — no RSA or DSA
- Key comment format: `| Purpose | Environment | User | YYYY.WW.Rev |`

### Role Meta (`meta/main.yml`)
- `license: MIT-0`
- `min_ansible_version: "2.14"`
- Platforms: Ubuntu jammy, noble

---

## Secrets & Vault

- Vault-encrypted files are committed to the repo (safe — content is encrypted)
- Plain vars and vault vars split into separate files under `group_vars/<group>/`:
  - `vars.yml` — plain variables
  - `vault.yml` — ansible-vault encrypted secrets
- `vault_password_file = ~/.ansible_vault_pass` set in `ansible.cfg` on Azir
- Vault pass file written to Azir by `secrets` role during bootstrap
- GitHub Actions workflows access vault password via repository secrets

---

## Inventory Conventions

- YAML format only — no INI inventory
- Three environments: `production`, `staging`, `dmz`
- Same group skeleton in every inventory:
  `docker_hosts`, `traefik_hosts`, `pi_hosts`, `vm_hosts`, `ansible_control`, `production`
- `ansible_host` and `ansible_user` set per-host in inventory file
- `group_vars/all.yml` sets `timezone` and `ansible_python_interpreter` per environment
- `ansible_control` group contains only Azir — tasks guarded by
  `when: "'ansible_control' in group_names"` are safe in any playbook

---

## In Progress

### `actions_runner` Role
- Installs and configures self-hosted GitHub Actions runner on target hosts
- Variables:
  - `actions_runner_version`: Runner version to install
  - `actions_runner_group`: Runner group name
  - `actions_runner_repo_url`: Repository URL to register with
- Tasks:
  - Download and extract runner archive
  - Create dedicated user (optional)
  - Configure runner with auth token
  - Install as systemd service
- Handler: `Restart actions_runner` to restart the service

---

## Roadmap & Future Work

### Docker Services Repo (`kylar514/docker-services`)
- Separate GitHub repo for all Docker Compose files and service configs
- Structure:
  - `core/` — traefik, adguard, adguardhome-sync (deploys to `traefik_hosts`)
  - `services/kennen/` — vaultwarden, arr-stack, torrent, omnitools, IT tools
  - `media/nasus/` — jellyfin (GPU passthrough, stays on NAS)
- Compose files deploy to `/opt/<service-name>/` on target hosts
- Workflows will trigger service deployments on this repo

### Planned — Ansible Roles
- `deploy_services` — copy compose files to `/opt/<service>/`, run `docker compose up -d --remove-orphans`
- `nfs_server` — configure NFS exports on Nasus for VM storage access
- `jellyfin` — manage Jellyfin container on Nasus (media_hosts group)

### Planned — Playbooks
- `deploy-core.yml` — rolling HA update for traefik_hosts with keepalived VIP handoff
  (`serial: 1`, pre-task lowers keepalived priority to release VIP, post-task reboots
  and restores priority — zero downtime updates)
- `deploy-services.yml` — non-HA service deployment to kennen

### Planned — Infrastructure
- `nasus` added to inventory as `media_hosts` group
- DMZ inventory populated — webhook receiver once DMZ network is configured
