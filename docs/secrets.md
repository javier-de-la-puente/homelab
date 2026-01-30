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

The private key must be available at `~/.config/sops/age/keys.txt` or via the `SOPS_AGE_KEY_FILE` environment variable.

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

Install the SOPS collection:
```bash
ansible-galaxy collection install community.sops
```

And ensure `sops` is in PATH.
