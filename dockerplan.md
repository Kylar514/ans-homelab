# Docker Compose Deployment Plan

## Overview

Ansible roles manage the Docker host environment and run `docker compose` on service
directories cloned from the `docker-services` repo. The `docker-services` CI workflow
calls `deploy-compose.yml` when compose files change, targeting only the affected hosts.

---

## /srv/docker Layout

```
/srv/docker/                          zeal:zeal        0755
  docker-services/                    zeal:dockersvc   2770  ← git repo
    core/                                                      → traefik_hosts
      traefik/
        compose.yml
        config/                       ← config-as-code, mounted :ro
    services/
      kennen/                                                  → kennen
        vaultwarden/
          compose.yml
  data/                               dockersvc:dockersvc 0770  ← container state
    <service>/                        created on first run via user: 2000:2000
  env/                                zeal:zeal        0700  ← Ansible-written secrets
    core/traefik.env                  zeal:zeal        0600
    services/kennen/vaultwarden.env
  globaldata/                         dockersvc:dockersvc 0770  ← NFS mountpoint (future)
```

### Data Directory Categories

| Category | Location | Owner | Managed by |
|----------|----------|-------|-----------|
| Config-as-code (read-only) | Inside repo clone | `zeal:dockersvc` | Git |
| Config-as-code (service-writable) | Inside repo clone | `zeal:dockersvc 2770` | Git + Ansible fixup |
| Container runtime state | `/srv/docker/data/<service>/` | `dockersvc:dockersvc` | Docker (created on first run) |
| Secrets | `/srv/docker/env/<path>.env` | `zeal:zeal 0600` | Ansible vault |
| Shared/media data | `/srv/docker/globaldata/` | `dockersvc:dockersvc` | NFS from Nasus (future) |

### compose file conventions
- All services use `user: "2000:2000"` (dockersvc uid:gid) — containers never run as root
- Config-as-code mounted `:ro` from inside the repo clone where possible
- Runtime state mounted from `/srv/docker/data/<service>/`
- Secrets loaded via `env_file: /srv/docker/env/<path>.env`
- Shared/media data will mount from `/srv/docker/globaldata/` once NFS is live

---

## Roles

### `docker_compose`

**Purpose**: Run `docker compose` commands on specific service directories.

**Variables**:
- `docker_compose_action`: string (default: `up -d --remove-orphans`)
- `docker_compose_services`: list of paths relative to `docker_compose_base_dir`
- `docker_compose_base_dir`: base path (default: `/srv/docker/docker-services`)

**Tasks**:
- Loop `docker_compose_services`
- Run `docker compose {{ docker_compose_action }}` in each directory
- `changed_when: false` — compose is idempotent, output parsing not worth the complexity

**Note**: Ansible user (zeal) runs compose directly. `user: 2000:2000` in each compose file
ensures containers run as dockersvc. No `become_user` override needed.

---

### `docker_env`

**Purpose**: Write `.env` files from vault variables to `/srv/docker/env/` on target hosts.

**Variables**:
- `docker_env_services`: list of `{path: 'core/traefik', template: 'traefik.env.j2'}`
- `docker_env_base_dir`: base path (default: `/srv/docker/env`)

**Tasks**:
- Ensure parent directory exists (`zeal:zeal 0700`)
- Template `.env` file to `{{ docker_env_base_dir }}/{{ item.path }}.env`
- Owner `zeal:zeal`, mode `0600`, `no_log: true`

**Templates**: Live in `roles/docker_env/templates/` — one `.env.j2` per service.
Added when a service is onboarded. Vault vars sourced from `group_vars/<group>/vault.yml`.

**When to run**: Manually via `deploy-env.yml` when rotating secrets. Never in compose CI.

---

### `docker_prune`

**Purpose**: Prune unused Docker resources.

**Variables** (all default `false`):
- `docker_prune_images`
- `docker_prune_containers`
- `docker_prune_networks`
- `docker_prune_volumes` — **never use in CI**, only manually with services confirmed running

**Tasks**: 4 conditional `command:` tasks, all `changed_when: false`.

---

## Playbooks

### `deploy-compose.yml`

Called by `docker-services` CI when compose files change. Always use `--limit`.

```bash
ansible-playbook playbooks/deploy-compose.yml \
  --extra-vars 'docker_compose_services=["core/traefik"]' \
  --limit traefik_hosts

ansible-playbook playbooks/deploy-compose.yml \
  --extra-vars 'docker_compose_action=pull docker_compose_services=["services/kennen/vaultwarden"]' \
  --limit kennen
```

### `deploy-env.yml`

Manual only. Run when rotating secrets. Does NOT restart services.

```bash
ansible-playbook playbooks/deploy-env.yml --limit kennen
ansible-playbook playbooks/deploy-env.yml --limit traefik_hosts
```

### `docker-prune.yml`

Manual only. All flags off by default.

```bash
ansible-playbook playbooks/docker-prune.yml \
  --extra-vars 'docker_prune_images=true docker_prune_networks=true' \
  --limit kennen
```

---

## `docker-services` Repo Structure

```
docker-services/
  core/                        → --limit traefik_hosts
    traefik/
      compose.yml
      config/
    adguard/
      compose.yml
    adguardhome-sync/
      compose.yml
  services/
    kennen/                    → --limit kennen
      vaultwarden/
        compose.yml
      arr-stack/
        compose.yml
      torrent/
        compose.yml
      omnitools/
        compose.yml
  .github/
    workflows/
      deploy.yml
  .gitignore
```

---

## CI Workflow (`docker-services/.github/workflows/deploy.yml`)

Trigger: push to `core/**` or `services/**`.

Logic:
1. `tj-actions/changed-files@v44` detects changed files
2. Shell step maps directory prefix → `--limit` target:
   - `core/*` → `traefik_hosts`
   - `services/kennen/*` → `kennen`
3. Runs `ansible-playbook deploy-compose.yml` with `--limit` + `--extra-vars`
4. Runs on `[self-hosted, azir]`

```yaml
name: Deploy Compose

on:
  push:
    paths:
      - 'core/**'
      - 'services/**'

jobs:
  deploy:
    runs-on: [self-hosted, azir]
    steps:
      - uses: actions/checkout@v4

      - name: Get changed files
        id: changed
        uses: tj-actions/changed-files@v44
        with:
          files: |
            core/**
            services/**

      - name: Deploy changed services
        run: |
          for file in ${{ steps.changed.outputs.all_changed_files }}; do
            dir=$(dirname "$file")
            case "$dir" in
              core/*)           limit="traefik_hosts" ;;
              services/kennen*) limit="kennen" ;;
              *) echo "Unknown path $dir, skipping"; continue ;;
            esac
            ansible-playbook playbooks/deploy-compose.yml \
              --vault-password-file ~/.ansible_vault_pass \
              --limit "$limit" \
              --extra-vars "docker_compose_services=[\"$dir\"]"
          done
```

---

## NFS / globaldata (Future)

`/srv/docker/globaldata/` is created as a placeholder by the `docker` role.
Once Nasus is added to inventory:

1. Build `nfs_server` role — configure NFS exports on Nasus
2. Build `nfs_client` role — mount `/srv/docker/globaldata/` on docker_hosts
3. Update UFW rules to allow NFS traffic (ports 2049, 111) between hosts
4. Update service compose files to reference `/srv/docker/globaldata/` for shared data

Services that will use globaldata: Jellyfin (media library), arr-stack (download targets),
any service needing shared state across multiple hosts.

---

## Onboarding a New Service

1. Add `compose.yml` to the correct path in `docker-services` repo
2. Use `user: "2000:2000"` in the compose file
3. Mount runtime state from `/srv/docker/data/<service>/` (Docker creates on first run)
4. Mount config-as-code from `./config/` inside the repo (`:ro` where possible)
5. If secrets needed: add `.env.j2` template to `roles/docker_env/templates/`, add entry
   to `docker_env_services` in the appropriate `group_vars`, run `deploy-env.yml`
6. Push to `docker-services` — CI deploys automatically via `deploy-compose.yml`
