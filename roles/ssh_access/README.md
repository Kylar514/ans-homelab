# ssh_access

Manage authorized_keys for SSH access.

## Variables

- `ssh_users`: List of users with `name` and `pubkeys`;

example

```
ssh_users:
  - name: zeal
    pubkeys:
      - homelab.pub
```

- Users lists are located in `inventories/production/group_vars/all/users_*.yml`
- Pubkeys are located in `inventories/production/files/*.pub`
