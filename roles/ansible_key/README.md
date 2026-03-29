# Files

Deploy Files to all regular users.

**This is a copy of copy-files role, built exclusively for bootstrapping
ansible**

## Variables

- Files src is always relative to `inventory_dir/files/`
- src, dest, owner, group are mandatory, mode is optional, defaulting to `0644`
- `ansible_key`
- Format:

```yaml
ansible_key:
  - src: 'bashrc'
    dest: '/etc/skel/.bashrc'
    owner: root
    group: root
    mode: 0644
```

- Files lists are located in
  `inventories/production/group_vars/ansible_control/ansible_key.yml`
- Files are located in `inventories/production/files/privatekey/*`
