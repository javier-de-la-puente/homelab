# Secrets Management

## Overview

Secrets are managed using [SOPS](https://github.com/getsops/sops) with [age](https://github.com/FiloSottile/age) encryption. This provides:

- **Encryption at rest:** Secrets are encrypted in git
- **Selective encryption:** Only sensitive fields are encrypted
- **Easy editing:** `sops` CLI decrypts in-place for editing

## Secrets Locations

| Path | Contents |
|------|----------|
| `flux/clusters/single-node/*/secrets.yaml` | Kubernetes secrets (data/stringData fields encrypted) |
| `ansible/group_vars/openwrt/secrets.sops.yml` | OpenWRT secrets (full file encrypted) |

## age Key

All secrets use the same age public key:
```
age1w8lcex3r5geeuxuvdfdvdjwm5y6djv7vwd7rqdj6awzpme8wc49qltgxgm
```

The private key must be available at `~/.sops/age-key.txt` (configured in `ansible/ansible.cfg`) or via the `SOPS_AGE_KEY_FILE` environment variable.

## SOPS Configuration

The `.sops.yaml` file defines encryption rules:

```yaml
creation_rules:
  # Kubernetes secrets - encrypt data/stringData fields only
  - path_regex: flux/.*\.yaml$
    encrypted_regex: ^(data|stringData)$
    age: age1w8lcex3r5geeuxuvdfdvdjwm5y6djv7vwd7rqdj6awzpme8wc49qltgxgm

  # OpenWRT secrets - encrypt entire file
  - path_regex: ansible/group_vars/openwrt/secrets\.sops\.yml$
    age: age1w8lcex3r5geeuxuvdfdvdjwm5y6djv7vwd7rqdj6awzpme8wc49qltgxgm
```

## Common Operations

### Viewing Secrets

```bash
# View decrypted content
sops -d ansible/group_vars/openwrt/secrets.sops.yml

# View specific K8s secret
sops -d flux/clusters/single-node/grafana/secrets.yaml
```

### Editing Secrets

```bash
# Opens in $EDITOR with decrypted content, re-encrypts on save
sops ansible/group_vars/openwrt/secrets.sops.yml
```

### Creating New Secrets

For Kubernetes secrets:
```bash
# Create unencrypted file first
cat > flux/clusters/single-node/myapp/secrets.yaml << 'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp
type: Opaque
stringData:
  password: supersecret
EOF

# Encrypt in place
sops -e -i flux/clusters/single-node/myapp/secrets.yaml
```

For Ansible secrets, copy the template and edit:
```bash
sops ansible/group_vars/openwrt/secrets.sops.yml
```

### Rotating Secrets

1. Edit the secret file with new values:
   ```bash
   sops ansible/group_vars/openwrt/secrets.sops.yml
   ```

2. Apply the changes:
   ```bash
   # For OpenWRT
   ansible-playbook playbooks/openwrt-full.yml -l router

   # For Kubernetes (Flux auto-applies)
   git add . && git commit -m "Rotate secrets" && git push
   ```

## OpenWRT Secrets

The `ansible/group_vars/openwrt/secrets.sops.yml` file contains:

| Key | Description |
|-----|-------------|
| `pppoe_username` | ISP PPPoE username |
| `pppoe_password` | ISP PPPoE password |
| `wifi_password` | Management WiFi password |
| `root_password_hash` | Router root password (hashed) |
| `he_tunnel.*` | Hurricane Electric IPv6 tunnel credentials |
| `wireguard.private_key` | WireGuard server private key |
| `wireguard.peers[].public_key` | VPN peer public keys |

### Generating Password Hash

```bash
# For root_password_hash
openssl passwd -6 'yourpassword'
```

### Generating WireGuard Keys

```bash
# Generate server keys
wg genkey | tee server_private | wg pubkey > server_public

# Generate peer keys
wg genkey | tee peer_private | wg pubkey > peer_public
```

## Kubernetes Secrets

Flux automatically decrypts SOPS-encrypted secrets using the age key configured in the cluster.

### Setting Up Flux SOPS

The age private key must be deployed to the cluster:

```bash
# Create secret with age key
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=$HOME/.config/sops/age/keys.txt

# Configure Flux to use it (in kustomization.yaml)
# decryption:
#   provider: sops
#   secretRef:
#     name: sops-age
```

## Backup and Recovery

### Backing Up the age Key

**Critical:** If you lose the age private key, all secrets become unrecoverable.

1. **Secure location:** Store in a password manager (1Password, Bitwarden)
2. **Offline backup:** Print or write down and store physically
3. **Multiple copies:** Keep backups in separate locations

### Recovering from Key Loss

If the age key is lost:
1. Generate a new age keypair
2. Update `.sops.yaml` with new public key
3. Re-create all secrets with new values
4. Re-encrypt with new key

## Troubleshooting

### "Failed to get the data key"

The age private key is not available. Check:
```bash
# Key file exists
cat ~/.config/sops/age/keys.txt

# Or set environment variable
export SOPS_AGE_KEY_FILE=/path/to/keys.txt
```

### "Could not find tree path"

The file path doesn't match any rule in `.sops.yaml`. Add a matching rule or check the path.

### Ansible Can't Decrypt

1. Install the SOPS collection:
   ```bash
   ansible-galaxy collection install community.sops
   ```

2. Ensure `sops` is in PATH

3. Verify the age key path in `ansible/ansible.cfg`:
   ```ini
   [community.sops]
   age_keyfile = ~/.sops/age-key.txt
   ```

4. Check that the `inventory/group_vars` symlink exists:
   ```bash
   ls -la ansible/inventory/group_vars
   # Should point to ../group_vars
   ```

## How Ansible SOPS Integration Works

The `community.sops.sops` vars plugin automatically decrypts SOPS-encrypted files in `group_vars/` and `host_vars/` directories.

### Configuration

The vars plugin is enabled in `ansible/ansible.cfg`:

```ini
[defaults]
vars_plugins_enabled = host_group_vars,community.sops.sops

[community.sops]
age_keyfile = ~/.sops/age-key.txt
```

### Important: Do NOT Use vars_files for SOPS Files

**Never load SOPS-encrypted files via `vars_files` in playbooks:**

```yaml
# WRONG - bypasses SOPS decryption!
vars_files:
  - ../group_vars/openwrt/secrets.sops.yml

# CORRECT - let the vars plugin handle it automatically
# (no vars_files entry for secrets.sops.yml)
```

The `vars_files` directive loads files as raw YAML without decryption. The SOPS vars plugin handles decryption automatically when files are in `group_vars/` or `host_vars/`.

### Directory Structure

The SOPS vars plugin looks for encrypted files in:
- `<inventory_dir>/group_vars/`
- `<playbook_dir>/group_vars/`

A symlink ensures the plugin finds the files:
```
ansible/inventory/group_vars -> ../group_vars
```

### Variable Precedence

When both `host_group_vars` and `community.sops.sops` plugins are enabled:
1. `host_group_vars` loads `.yml` files (including `.sops.yml` as raw YAML)
2. `community.sops.sops` loads and decrypts `.sops.yml` files
3. **Last defined wins** - decrypted values override encrypted strings
