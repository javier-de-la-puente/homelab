# Homelab

Infrastructure as Code for my homelab network and Kubernetes cluster.

## Overview

This repository manages:

- **OpenWRT Router** - Raspberry Pi 4 running OpenWRT with VLANs, WireGuard VPN, and firewall
- **Kubernetes Cluster** - Single-node K8s with Cilium CNI and Flux GitOps
- **Applications** - Nextcloud, Grafana, Prometheus, Loki, and more

## Architecture

```
Internet
    │
[Pi 4 OpenWRT] ─── 10.50.1.1 (Management)
    │
[Managed Switch]
    │
    ├── VLAN 10/11 ─── Legacy Services (192.168.10-11.x)
    ├── VLAN 20 ─── Trusted Devices (10.50.20.x)
    ├── VLAN 30 ─── IoT (10.50.30.x)
    ├── VLAN 40 ─── Guest (10.50.40.x)
    ├── VLAN 50 ─── WireGuard VPN (10.50.50.x)
    └── VLAN 60 ─── Kubernetes (10.50.60.x)
              │
         [firebata] ─── K8s Node (10.50.60.100)
              │
         LoadBalancer Pool (10.50.60.200-250)
```

## Quick Start

### Router Setup

1. Flash OpenWRT to Raspberry Pi 4 SD card
2. Connect Pi to laptop, run bootstrap:
   ```bash
   cd ansible
   ansible-playbook playbooks/openwrt-bootstrap.yml -i inventory/bootstrap/openwrt.yml
   ```
3. Connect to new network, run full config:
   ```bash
   ansible-playbook playbooks/openwrt-full.yml -i inventory/hosts.yml -l router
   ```

### Kubernetes Setup

```bash
cd ansible
ansible-playbook playbooks/k8s-node.yml -i inventory/hosts.yml -l firebata
```

Flux automatically deploys applications from `flux/clusters/single-node/`.

## Repository Structure

```
.
├── ansible/                  # Infrastructure provisioning
│   ├── inventory/           # Host inventories
│   ├── group_vars/          # Variables by group
│   ├── roles/               # Ansible roles
│   └── playbooks/           # Runnable playbooks
├── flux/                     # GitOps manifests
│   └── clusters/single-node/
│       ├── cilium/          # CNI configuration
│       ├── gateway/         # Ingress gateway
│       ├── longhorn/        # Storage
│       ├── nextcloud/       # File sharing
│       ├── grafana/         # Dashboards
│       ├── prometheus/      # Metrics
│       └── loki/            # Logs
├── pxe/                      # PXE boot provisioning
├── docs/                     # Documentation
│   ├── network.md           # Network architecture
│   ├── router.md            # OpenWRT management
│   └── secrets.md           # Secrets handling
└── .sops.yaml               # SOPS encryption config
```

## Documentation

- [Network Architecture](docs/network.md) - VLANs, subnets, firewall zones
- [Router Management](docs/router.md) - OpenWRT setup and configuration
- [Secrets Management](docs/secrets.md) - SOPS+age encryption
- [Ansible Guide](ansible/README.md) - Playbooks and roles

## Key Technologies

| Component | Technology |
|-----------|------------|
| Router | OpenWRT on Raspberry Pi 4 |
| Kubernetes | kubeadm, single-node |
| CNI | Cilium with L2 LoadBalancer |
| GitOps | Flux CD |
| Storage | Longhorn |
| Secrets | SOPS + age |
| VPN | WireGuard |

## Secrets

All secrets are encrypted with SOPS+age. To edit:

```bash
# OpenWRT secrets
sops ansible/group_vars/openwrt/secrets.sops.yml

# Kubernetes secrets
sops flux/clusters/single-node/grafana/secrets.yaml
```

See [docs/secrets.md](docs/secrets.md) for setup.
