# Files

Deploy Files to all regular users.

## Variables

- Files src is always relative to `inventory_dir/files/`
- src, dest, owner, group are mandatory, mode is optional, defaulting to `0644`
- `all_files` & `group_files` are added together
- Format:

```yaml
all_files:
  - src: 'bashrc'
    dest: '/etc/skel/.bashrc'
    owner: root
    group: root
    mode: 0644

group_files:
  - src: 'vimrc'
    dest: '/etc/skel/.vimrc'
    owner: root
    group: root
```

- Files lists are located in
  `inventories/production/group_vars/all/filetransfer.yml`
- Files are located in `inventories/production/files/files/*`
