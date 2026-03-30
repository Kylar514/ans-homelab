# k3s

Installs and configures a k3s cluster. Delegates to server tasks for control
plane nodes (kaijus) and agent tasks for worker nodes (corps).

## Variables

- Group-specific config in `inventories/production/group_vars/k3s/k3s.yml`

```yaml
k3s_version: 'v1.32.2+k3s1'
k3s_server_ip: '10.0.40.8'
k3s_token: "{{ lookup('community.hashi_vault.hashi_vault', ...) }}"
```

- `k3s_token` is pulled from Vault at runtime via AppRole — see vault role
- `k3s_server_ip` points to kaiju-01, update to VIP when going HA

## UFW

Port 6443 must be open on control plane nodes:

- `inventories/production/group_vars/kaijus/ufw.yml`

## Post-Install (manual, one-time)

Copy kubeconfig to your desktop and azir-01:

```bash
ssh kaiju-01 "sudo cat /etc/rancher/k3s/k3s.yaml" > ~/.kube/config
sed -i 's/127.0.0.1/10.0.40.8/g' ~/.kube/config
```

## Notes

- k3s ships with Traefik as the default ingress controller
- Agents wait for the server API on 6443 before attempting to join
- Idempotent via `creates:` — won't reinstall if service file exists
