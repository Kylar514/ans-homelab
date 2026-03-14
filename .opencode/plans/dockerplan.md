# Docker Compose Deployment Plan

## Overview

Create Ansible roles to run `docker compose` and prune Docker resources, triggered from docker-services GitHub Actions workflow when compose files change.

## Repository Structure

- **docker-services repo**: cloned to `/home/zeal/docker-services/` on each host
- **Structure**: `infra/`, `services/`, `media/` directories containing service subdirs with compose files
- **Runners**: 3 self-hosted runners on Azir, 1 connected to docker-services

## Integration Flow

```
docker-services workflow (detects changed compose file)
  → runs on Azir runner
  → ansible-playbook playbooks/deploy-compose.yml --extra-vars 'docker_compose_services=["<path>"]'
```

## Roles to Create

### 1. docker_compose

**Purpose**: Run docker compose commands on specific services

**Variables**:
- `docker_compose_action`: string (default: `up -d --remove-orphans`)
- `docker_compose_services`: list of paths relative to `/home/zeal/docker-services/`

**Example values**:
- `docker_compose_services: ['infra/traefik', 'services/arr-stack']`
- `docker_compose_action: 'pull'` (for updating images)

**Tasks**:
- Loop through `docker_compose_services`
- Run `docker compose {{ docker_compose_action }}` in each directory
- Use `changed_when: false` or analyze output properly
- Set workdir to `/home/zeal/docker-services/{{ item }}`

### 2. docker_prune

**Purpose**: Prune unused Docker resources

**Variables**:
- `docker_prune_images`: boolean (default: `false`)
- `docker_prune_containers`: boolean (default: `false`)
- `docker_prune_networks`: boolean (default: `false`)
- `docker_prune_volumes`: boolean (explicit opt-in, default: `false`)

**Tasks**:
- Prune images if enabled
- Prune containers if enabled
- Prune networks if enabled
- Prune volumes only if explicitly enabled (guard with warning)
- Use `docker image prune`, `docker container prune`, etc.

### 3. (Optional) docker_update_all

**Purpose**: Pull latest images for all compose files across all hosts

**Variables**:
- `docker_compose_services`: list of all services (or detect all subdirs)
- `docker_compose_action: 'pull'`

**Tasks**:
- Run `docker compose pull` on all service directories

## Playbook to Create

### playbooks/deploy-compose.yml

```yaml
# SPDX-License-Identifier: MIT-0
---
# Deploy docker compose services
# Usage:
#   ansible-playbook playbooks/deploy-compose.yml --extra-vars 'docker_compose_services=["infra/traefik"]'
#   ansible-playbook playbooks/deploy-compose.yml --extra-vars 'docker_compose_action=pull'

- name: Deploy Docker Compose Services
  hosts: docker_hosts
  become: true
  roles:
    - docker_compose
```

## GitHub Actions (in docker-services repo)

```yaml
name: Deploy Compose

on:
  push:
    paths:
      - 'infra/**'
      - 'services/**'
      - 'media/**'

jobs:
  deploy:
    runs-on: [self-hosted, azir]
    steps:
      - name: Get changed compose files
        id: changed
        uses: tj-actions/changed-files@v44
        with:
          files: |
             infra/**
             services/**
             media/**

      - name: Extract service paths
        run: |
          SERVICES=$(echo '${{ steps.changed.outputs.all_changed_files }}' | \
            xargs -I{} dirname {} | sort -u)
          echo "services=$SERVICES" >> $GITHUB_OUTPUT

      - name: Run Ansible
        run: |
          ansible-playbook playbooks/deploy-compose.yml \
            --vault-password-file ~/.ansible_vault_pass \
            --extra-vars "docker_compose_services=${{ steps.changed.outputs.all_changed_files }}"
```

## Notes

- Uses `docker-compose-plugin` (already installed) - commands are `docker compose` not `docker-compose`
- Each host clones full docker-services repo to `/home/zeal/docker-services/`
- Role uses community.docker.docker_compose module OR shell with `docker compose` command
- Prune volumes must be explicit opt-in to avoid accidental data loss

## Convention Checklist

- [ ] License header: `# SPDX-License-Identifier: MIT-0`
- [ ] Follow role variable naming: `docker_compose_*`, `docker_prune_*`
- [ ] Use `become: true` at play level
- [ ] Use `loop:` not `with_items:`
- [ ] `changed_when: false` on command tasks that only gather facts
- [ ] Handler names: `Restart docker_compose` (if needed)
- [ ] Template first line: `# Managed by Ansible - do not edit manually`
