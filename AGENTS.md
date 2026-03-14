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
| **Ansible Test** | Pull request to `master` / push to `test` | Syntax check + dry run (`--check --diff`) against production and staging inventories |
| **Prod CI/CD** | Push to `master` branch | SSH into prod and staging hosts, pull `master`, run `site.yml` against both |

### Workflow Behavior

- **Ansible Test**: Runs `ansible-playbook --syntax-check` + `ansible-playbook --check --diff` against both production and staging inventories. Does NOT execute any changes
- **Prod CI/CD**: Targets all production hosts AND staging host, uses `inventories/production/inventory.yml` and `inventories/staging/inventory.yml`

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
ansible-playbook playbooks/bootstrap.yml --private-key ~/.ssh/homelab \
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

# Deploy compose services (called by docker-services CI, or manually)
ansible-playbook playbooks/deploy-compose.yml \
  --extra-vars 'docker_compose_services=["core/traefik"]' \
  --limit traefik_hosts

# Write .env secret files from vault (manual — run when rotating secrets)
ansible-playbook playbooks/deploy-env.yml --limit kennen

# Prune unused Docker resources (manual only — never volumes in CI)
ansible-playbook playbooks/docker-prune.yml \
  --extra-vars 'docker_prune_images=true docker_prune_networks=true' \
  --limit kennen
```

---

## Repository Structure

```
ans-homelab/
  .github/
    workflows/                 # GitHub Actions workflows
  ansible.cfg                 # private_key_file, inventory, vault_password_file, remote_user
  AGENTS.md                   # This file
  dockerplan.md               # Docker Compose deployment design spec
  playbooks/
    site.yml                  # Master converge — imports all playbooks in order
    base.yml                  # SSH keys, packages, upgrades — targets all hosts
    bootstrap.yml             # Azir control node setup — run once after Terraform
    ufw.yml                   # Firewall rules — SSH always unconditionally allowed
    docker.yml                # Docker CE + dockersvc user/group + /srv/docker dirs
    vip_failover.yml          # Keepalived VIP failover — traefik_hosts only
    maintain.yml              # Self-maintenance (keychain reload, etc.)
    deploy-compose.yml        # Deploy docker compose services — called by docker-services CI
    deploy-env.yml            # Write .env files from vault — manual, secret rotation only
    docker-prune.yml          # Prune unused Docker resources — manual only
  roles/
    ssh_access/               # authorized_keys managed via Jinja2 template
    sys_packages/             # Base packages + ansible_control conditional packages
    unattended_upgrades/      # Automatic security updates
    ufw/                      # UFW firewall — SSH rule is unconditional task
    docker/                   # Docker CE install, dockersvc group/user, /srv/docker dirs
    vip_failover/             # Keepalived config template + restart handler
    clone_repos/              # Generic git clone/update role with optional permissions fixup
    secrets/                  # Write vault secrets to disk (any host)
    keychain/                 # Configure keychain + SSH key loading on login
    timezone/                 # Set system timezone
    actions_runner/           # Self-hosted GitHub Actions runner (Azir only)
    docker_compose/           # Run docker compose commands on docker_hosts
    docker_env/               # Write .env files from vault to /srv/docker/env/
    docker_prune/             # Prune unused Docker resources
  inventories/
    production/               # azir, ryze, zilean, kennen
    staging/                  # heimerdinger
    dmz/                      # Future use
```

---

## /srv/docker Layout (docker_hosts)

Ansible creates the top-level structure. Service data dirs are created by Docker on first run
(containers run as `user: 2000:2000` in compose, so bind-mount dirs are owned by `dockersvc`).

```
/srv/docker/                          zeal:zeal        0755  top level
  docker-services/                    zeal:dockersvc   2770  git repo (clone_repos)
    core/                                                     → traefik_hosts services
    services/                                                 → per-host services
  data/                               dockersvc:dockersvc 0770  container bind-mount state
    <service>/                                               created by Docker on first run
  env/                                zeal:zeal        0700  Ansible-written .env secrets
    core/traefik.env                  zeal:zeal        0600  per-service env files
    services/kennen/vaultwarden.env
  globaldata/                         dockersvc:dockersvc 0770  NFS mountpoint (future)
```

### Ownership rationale
- `docker-services/` is `zeal:dockersvc 2770` — zeal owns (git works normally), dockersvc group
  can read/write (service-writable config files), setgid propagates group on new files/dirs.
  Security note: dockersvc can modify compose files, but compose only runs after Ansible git pull
  which overwrites them — exploitability window is effectively zero.
- `data/` is `dockersvc:dockersvc` — containers write here. Zeal (in dockersvc group) can inspect.
- `env/` is `zeal:zeal 0700` — only zeal reads/writes. Docker passes vars into containers at
  runtime via `env_file:` in compose; dockersvc never directly reads these files.
- `globaldata/` placeholder for future NFS mount from Nasus.

### compose file conventions
- All services use `user: "2000:2000"` (dockersvc uid:gid) — containers never run as root
- Config-as-code (read-only) mounted `:ro` from inside the repo clone
- Runtime state mounted from `/srv/docker/data/<service>/`
- Secrets loaded via `env_file: /srv/docker/env/<path>.env`
- Shared/media data will mount from `/srv/docker/globaldata/` (once NFS is live)

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
- Service `.env` files written to `/srv/docker/env/` by `deploy-env.yml` — never committed

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

## Roadmap & Future Work

### Next — `docker-services` Repo (`kylar514/docker-services`)
- Separate GitHub repo for all Docker Compose files and service configs
- Cloned to `/srv/docker/docker-services/` on all `docker_hosts` by `clone_repos`
- Structure:
  - `core/` — traefik, adguard, adguardhome-sync (deploys to `traefik_hosts`)
  - `services/kennen/` — vaultwarden, arr-stack, torrent, omnitools, IT tools
- CI workflow maps path prefix → `--limit` target, calls `deploy-compose.yml`
- See `dockerplan.md` for full design spec

### Planned — Ansible Roles
- `nfs_server` — configure NFS exports on Nasus for shared storage
- `nfs_client` — mount Nasus NFS share at `/srv/docker/globaldata/` on docker_hosts

### Planned — Infrastructure
- `nasus` added to inventory as `media_hosts` + `nfs_server` group (after Nasus migration complete)
- DMZ inventory populated — webhook receiver once DMZ network is configured
