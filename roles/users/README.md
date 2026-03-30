# users

Configure users

## Variables

- All rules should be set through the variable `all_users` & `group_users` are
  combined together
- Format:

```yaml
all_users:
  - name: ansible
    group: ansible
    shell: bash
    sudo: true
    groups: # optional additional groups
      - 1
      - 2

group_users_rules:
  - name: k3s
    group: k3s
    shell: bash
    sudo: true
```

- Global users can be set in `inventories/production/all/users.yml`
- Group-specific users can be set in
  `inventories/production/groupname/users.yml`
