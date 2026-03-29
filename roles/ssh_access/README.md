# ssh_access

Manage authorized_keys for SSH access.

## Variables

- `ssh_users`: List of users with `name` and `pubkeys`;

example

```yaml
ssh_users:
  - name: zeal
    pubkeys:
      - homelab.pub
```

- Users lists are located in `inventories/production/group_vars/all/sshkey.yml`
  & `inventories/production/group_vars/group/sshkey.yml`
- Pubkeys are located in `inventories/production/files/pubkeys/*.pub`
