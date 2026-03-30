# vault

Installs and configures HashiCorp Vault on the ansible_control node (azir-01).
Uses Raft integrated storage. Exposes UI and API on port 8200.

> This homelab is intentionally security hardened, partly for the challenge,
> partly for the love of the game, partly to demonstrate capability across the
> full stack, and partly to find the weaknesses. Design decisions prioritize
> security principles over convenience where possible.

## Dependencies

- `community.hashi_vault` collection must be installed on the control node

## Variables

- Group-specific config in
  `inventories/production/group_vars/ansible_control/vault.yml`

```yaml
vault_addr: 'http://10.0.40.11:8200'
```

## Post-Install (manual, one-time)

After first deploy via `bootstrap.yml`, Vault must be manually initialized:

```bash
export VAULT_ADDR=http://localhost:8200
vault operator init -key-shares=1 -key-threshold=1
```

Save the unseal key in your password manager. Then unseal:

```bash
vault operator unseal
```

## Secrets Engine Setup (manual, one-time)

```bash
vault secrets enable -path=secret kv-v2
```

## AppRole Setup (manual, one-time)

```bash
vault policy write ansible - << 'EOF'
path "secret/data/*" {
  capabilities = ["read", "list"]
}
EOF

vault auth enable approle

vault write auth/approle/role/ansible \
  token_policies="ansible" \
  token_ttl=1h \
  token_max_ttl=4h

vault read auth/approle/role/ansible/role-id
vault write -f auth/approle/role/ansible/secret-id
```

## GitHub Actions

Add to repository settings:

| Type     | Key               | Value                   |
| -------- | ----------------- | ----------------------- |
| Secret   | `VAULT_SECRET_ID` | secret_id from above    |
| Variable | `VAULT_ROLE_ID`   | role_id from above      |
| Variable | `VAULT_ADDR`      | `http://localhost:8200` |

## Accessing Vault

No persistent users or tokens exist by design. To gain access:

```bash
# 1. generate root token using unseal key
vault operator generate-root -init

# 2. provide unseal key when prompted
vault operator generate-root -nonce=<nonce>

# 3. decode the encoded token with the OTP from step 1
vault operator generate-root -decode=<encoded-token> -otp=<otp>

# 4. login
vault login

# 5. revoke when done
vault token revoke -self
```

## Security Model

```
Unseal key    →  stored offline in password manager, never touches disk
Root token    →  does not exist permanently, generated via unseal key when needed
AppRole       →  read-only CI access, token TTL 1h max 4h
Userpass      →  disabled, break-glass via unseal key only
```

## Backups

```bash
vault operator raft snapshot save /backup/vault-snapshot.snap
```

## Notes

- Vault is managed via `bootstrap.yml`
- Unseal required after every reboot (make a script using a password manager)
- Root token should be revoked immediately after any administrative task
- No persistent admin users by design, least privilege, break-glass model
