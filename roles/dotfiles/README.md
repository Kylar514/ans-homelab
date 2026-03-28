# dotfiles

Deploy dotfiles to all regular users.

## Variables

- `dotfiles`: List of files with src and dest;

```yaml
dotfiles:
  - src: vimrc
    dest: .vimrc
  - src: bashrc
    dest: .bashrc
```

- Dotfiles lists are located in
  `inventories/production/group_vars/all/dotfiles.yml`
- Dotfiles are located in `inventories/production/files/dotfiles/*`
