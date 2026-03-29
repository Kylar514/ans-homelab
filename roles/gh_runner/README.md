# gh_runner

Configure Github Runner

## Variables

- Format:

```yaml
runner_user: ansible
runner_work_dir: /opt/gh-runner
runner_repo_url: 'https://github.com/kylar514/ans-homelab'
runner_name: '{{ inventory_hostname }}'
runner_repo_owner: kylar514
runner_repo_name: ans-homelab
```

- Group-specific rules can be set in
  `inventories/production/ansible_control/gh_runner.yml`

- PAT is stored in `vault.yml`
- Update PAT before running!
