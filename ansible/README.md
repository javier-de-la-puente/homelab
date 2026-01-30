# Ansible Configuration

This directory contains Ansible playbooks and roles for managing the homelab infrastructure.

## Directory Structure

```
ansible/
├── inventory/
│   ├── hosts.yml              # Production inventory (all hosts)
│   └── bootstrap/
│       └── openwrt.yml        # Bootstrap inventory (fresh OpenWRT at 192.168.1.1)
├── group_vars/
│   └── openwrt/
│       ├── network.yml        # VLAN definitions, subnets
│       ├── firewall.yml       # Zone policies, port forwards
│       ├── dhcp.yml           # DHCP pools, static reservations
│       └── secrets.sops.yml   # PPPoE, WireGuard, etc. (SOPS encrypted)
├── roles/
│   ├── openwrt/               # OpenWRT router configuration
│   ├── storage/               # Disk formatting and Longhorn setup
│   ├── containerd/            # Container runtime
│   ├── kubernetes_prereqs/    # K8s system prerequisites
│   ├── kubernetes/            # K8s cluster initialization
│   ├── cilium/                # CNI plugin
│   └── flux/                  # GitOps automation
├── playbooks/
│   ├── openwrt-bootstrap.yml  # Stage 1: Basic router connectivity
│   ├── openwrt-full.yml       # Stage 2: Full router configuration
│   └── k8s-node.yml           # Kubernetes node setup
├── site.yaml                  # Legacy: K8s bootstrap (use playbooks/ instead)
└── add-storage.yaml           # Add storage disks to existing cluster
```

## Prerequisites

- Ansible 2.14+
- Python 3.8+
- SOPS and age for secrets management
- SSH access to target hosts

Install required collections:
```bash
ansible-galaxy collection install community.general community.sops ansible.posix
```

## Playbooks

### OpenWRT Router

**Prerequisites** (fresh OpenWRT install):
```bash
# Share internet from laptop to router (connect laptop eth to router LAN)
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o wlp0s20f3 -j MASQUERADE

# On router: set date, switch opkg to HTTP, install Python3
ssh root@192.168.1.1
date -s '2026-01-29 12:00:00'
sed -i 's|https://|http://|g' /etc/opkg/distfeeds.conf
opkg update && opkg install python3
```

**Stage 1 - Bootstrap** (fresh install at 192.168.1.1):
```bash
ansible-playbook playbooks/openwrt-bootstrap.yml -i inventory/bootstrap/openwrt.yml
```

This configures:
- System hostname and timezone
- PPPoE WAN connection
- Management WiFi (backup access)
- New management IP: 10.50.1.1

**Stage 2 - Full Configuration** (router at 10.50.1.1):
```bash
# Update inventory to use 10.50.1.1 first, then:
ansible-playbook playbooks/openwrt-full.yml -i inventory/bootstrap/openwrt.yml
```

This configures:
- All VLANs (1, 10, 11, 20, 30, 40, 50, 60)
- Firewall zones and inter-VLAN policies
- DHCP pools and static reservations
- WireGuard VPN
- IPv6 tunnel
- Port forwards

### Kubernetes Node

**Full node bootstrap:**
```bash
ansible-playbook playbooks/k8s-node.yml -i inventory/hosts.yml -l firebata
```

**Add storage only:**
```bash
ansible-playbook add-storage.yaml -i inventory/hosts.yml -l firebata
```

## Inventory

### Production Inventory (`inventory/hosts.yml`)

Contains all hosts after initial setup:

| Host | Group | IP | Purpose |
|------|-------|-----|---------|
| router | openwrt | 10.50.1.1 | OpenWRT router |
| firebata | k8s | 10.50.60.100 | Kubernetes node |

### Bootstrap Inventory (`inventory/bootstrap/openwrt.yml`)

Used only for fresh OpenWRT installations:
- IP: 192.168.1.1 (OpenWRT default)
- No password (fresh install)

## Secrets

Secrets are stored in `group_vars/openwrt/secrets.sops.yml` and encrypted with SOPS+age. The `community.sops.sops` vars plugin automatically decrypts them at runtime.

**Prerequisites:**
- Age private key at `~/.sops/age-key.txt`
- Symlink: `inventory/group_vars -> ../group_vars`

**Edit secrets:**
```bash
sops group_vars/openwrt/secrets.sops.yml
```

**View secrets:**
```bash
sops -d group_vars/openwrt/secrets.sops.yml
```

Secrets include:
- PPPoE credentials (`pppoe_username`, `pppoe_password`)
- WAN MAC address (`wan_macaddr`)
- WiFi password (`wifi_password`)
- WireGuard keys (`wireguard.*`)
- Hurricane Electric tunnel credentials (`he_tunnel.*`)
- OVH DDNS credentials (`ovh_ddns.*`)

**Important:** Do NOT add `secrets.sops.yml` to `vars_files` in playbooks - the vars plugin handles decryption automatically.

See [docs/secrets.md](../docs/secrets.md) for details.

## Roles

### openwrt

Configures the OpenWRT router. Supports two stages:
- `bootstrap`: Minimal config for fresh install
- `full`: Complete configuration

### storage

Formats and mounts storage disks, configures Longhorn.

### containerd

Installs and configures containerd runtime.

### kubernetes_prereqs

System configuration for Kubernetes (kernel modules, sysctl, swap).

### kubernetes

Initializes Kubernetes cluster with kubeadm.

### cilium

Installs Cilium CNI with kube-proxy replacement.

### flux

Bootstraps Flux GitOps from GitHub.

## Variables

### Network Variables (`group_vars/openwrt/network.yml`)

- `vlans`: List of VLAN definitions (id, name, subnet, gateway)
- `wifi_ssid`, `wifi_password`: Management WiFi
- `wan_interface`, `lan_interface`: Physical interfaces

### Firewall Variables (`group_vars/openwrt/firewall.yml`)

- `firewall_zones`: Zone definitions and policies
- `firewall_forwardings`: Inter-zone forwarding rules
- `firewall_rules`: Additional firewall rules
- `port_forwards`: DNAT rules for external access

### DHCP Variables (`group_vars/openwrt/dhcp.yml`)

- `dns_servers`: Upstream DNS servers
- `local_domain`: Local DNS domain
- `static_hosts`: DHCP reservations by VLAN

## Troubleshooting

### SSH Connection Issues

For fresh OpenWRT:
```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@192.168.1.1
```

### SOPS Decryption Failures

Ensure age key is available at the path configured in `ansible.cfg`:
```bash
# Default location (configured in ansible.cfg)
~/.sops/age-key.txt

# Or set via environment variable
export SOPS_AGE_KEY_FILE=~/.sops/age-key.txt
```

Verify the inventory symlink exists:
```bash
ls -la inventory/group_vars
# Should show: group_vars -> ../group_vars
```

### Ansible Module Not Found

Install required collections:
```bash
ansible-galaxy collection install -r requirements.yml
```
