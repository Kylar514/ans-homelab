# ufw

Configure UFW firewall rules.

## Variables

- All rules should be set through the variable `all_ufw_rules` &
  `group_ufw_rules` are combined together
- Format:

```yaml
all_ufw_rules:
  - port:   The port number to allow/deny
    proto:  Protocol ('tcp' or 'udp'), defaults to 'tcp'
    rule:   'allow' or 'deny', defaults to 'allow'

group_ufw_rules:
  - port:   The port number to allow/deny
    proto:  Protocol ('tcp' or 'udp'), defaults to 'tcp'
    rule:   'allow' or 'deny', defaults to 'allow'
```

- Global rules can be set in `inventories/production/all/ufw.yml`
- Group-specific rules can be set in `inventories/production/groupname/ufw.yml`
- All

## Drift Prevention

- **UFW is imperative and stateful**:
  - Ansible can add rules, but removing rules in Ansible **does not
    automatically remove them on the server**, which can lead to configuration
    drift.
- **Applying a "deny all" rule can be risky**:
  - Setting a blanket deny can cause brief downtime and may be dangerous if a
    playbook only partially completes.
- **Mitigation plan**:
  - Use Grafana to monitor the UFW rule state on VMs.
  - Ideally, users should set any existing rules to deny if they are enabled in
    the variables.
  - This role and Ansible repo assume that Ansible is managing **all UFW rules
    from installation onward**.
  - Monitoring ensures that mistakes or drift are detected quickly.
