# sys_packages

Install base packages based on group

## Variables

- All rules should be set through the variable `all_packages` & `group_packages`
  are combined together
- Format:

```yaml
all_packages:
  - 1
  - 2
  - 3

group_packages:
  - 1
  - 2
  - 3
```

- Global rules can be set in `inventories/production/all/packages.yml`
- Group-specific rules can be set in
  `inventories/production/groupname/packages.yml`
