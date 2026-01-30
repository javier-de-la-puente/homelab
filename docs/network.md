# Network Architecture

## Overview

The homelab network uses a VLAN-based architecture with a Raspberry Pi 4 running OpenWRT as the router. The IP addressing scheme uses `10.50.x.0/24` where the VLAN ID matches the third octet (e.g., VLAN 20 = 10.50.20.x). Legacy services remain on `192.168.x.x` to avoid disruption.

## Network Topology

```
Internet
    │
    ▼
[ISP Modem/ONT]
    │
    ▼ (eth1 - WAN via VLAN 20 PPPoE)
[Pi 4 OpenWRT Router] ─── 10.50.1.1
    │ (eth0 - LAN trunk)
    │ (wlan0 - Management WiFi)
    │
    ▼
[Zyxel GS1200-8] ─── 10.50.1.2
    │
    ├── Port 1 ─── Trunk (all VLANs tagged) ─── Router
    ├── Port 2 ─── PVID=10,11 ─── Legacy services (192.168.10.x)
    ├── Port 3 ─── PVID=60 ─── servers (10.50.60.x)
    ├── Port 4 ─── PVID=60 ─── K8s firebata (10.50.60.x)
    ├── Port 5 ─── PVID=20 ─── Main AP (Trusted)
    ├── Port 6 ─── PVID=20 ─── Trusted device
    ├── Port 7 ─── PVID=30 ─── IoT hub
    └── Port 8 ─── PVID=40 ─── Guest AP
```

## VLANs and Subnets

| VLAN ID | Name | Subnet | Gateway | Purpose |
|---------|------|--------|---------|---------|
| 1 | Management | 10.50.1.0/24 | 10.50.1.1 | Router, switches, emergency access |
| 10 | Legacy Services | 192.168.10.0/24 | 192.168.10.1 | nginx, Minecraft, Matrix (keep existing) |
| 11 | Legacy Aux | 192.168.11.0/24 | 192.168.11.1 | t8plus, RPi5 (keep existing) |
| 20 | Trusted | 10.50.20.0/24 | 10.50.20.1 | Personal devices, main WiFi |
| 30 | IoT | 10.50.30.0/24 | 10.50.30.1 | Smart home devices (isolated) |
| 40 | Guest | 10.50.40.0/24 | 10.50.40.1 | Guest WiFi (internet only) |
| 50 | WireGuard | 10.50.50.0/24 | 10.50.50.1 | VPN clients (virtual) |
| 60 | Kubernetes | 10.50.60.0/24 | 10.50.60.1 | K8s nodes and LoadBalancer IPs |

## IP Allocations

### Management VLAN (10.50.1.0/24)

| Host | IP | Notes |
|------|-----|-------|
| Router | 10.50.1.1 | OpenWRT gateway |
| GS1200-8 | 10.50.1.2 | 8-port managed switch |
| GS1200-5 | 10.50.1.3 | 5-port managed switch |
| DHCP Pool | 10.50.1.100-249 | Dynamic clients |

### Kubernetes VLAN (10.50.60.0/24)

| Resource | IP/Range | Notes |
|----------|----------|-------|
| firebata | 10.50.60.100 | K8s control plane + workloads |
| Future nodes | 10.50.60.10-99 | Expansion |
| LoadBalancer Pool | 10.50.60.200-250 | Cilium L2 announcements |
| Pod CIDR | 10.244.0.0/16 | Internal (CNI) |
| Service CIDR | 10.96.0.0/12 | Internal (K8s) |

### Legacy Services (192.168.10.0/24)

| Host | IP | Notes |
|------|-----|-------|
| nginx | 192.168.10.100 | Reverse proxy, port forwards |

### Legacy Aux (192.168.11.0/24)

| Host | IP | Notes |
|------|-----|-------|
| t8plus | 192.168.11.2 | SSH server (external port 22) |
| RPi5 | 192.168.11.3 | |

### WireGuard VPN (10.50.50.0/24)

| Peer | IP | Notes |
|------|-----|-------|
| Server | 10.50.50.1 | OpenWRT router |
| thinkpadpersonal | 10.50.50.2 | |
| doctormorales | 10.50.50.3 | |

## Firewall Zones

| Zone | VLANs | Can Reach | Notes |
|------|-------|-----------|-------|
| lan | 1, 20, 60 | wan, all internal | Full access |
| legacy | 10, 11 | wan, lan | Legacy services |
| iot | 30 | wan only | Isolated, internet only |
| guest | 40 | wan only | Fully isolated |
| vpn | 50 | lan, legacy | VPN clients |
| wan | - | - | Inbound blocked except forwards |

### Inter-VLAN Rules

- **K8s (60) → IoT (30):** Allow (Home Assistant controls devices)
- **K8s (60) ↔ Legacy (10, 11):** Allow (services communicate)
- **IoT (30) → K8s/Legacy:** Block (can't initiate connections)
- **Guest (40) → anything internal:** Block (internet only)
- **Trusted (20) → K8s/Legacy:** Allow (access services)

## Port Forwards

| Service | External Port | Destination | Notes |
|---------|---------------|-------------|-------|
| HTTP | 80 | 192.168.10.100:80 | nginx |
| HTTPS | 443 | 192.168.10.100:443 | nginx |
| SSH | 22 | 192.168.11.2:22 | t8plus |
| SSH | 22222 | 10.50.60.100:22 | firebata |
| Minecraft | 25565 | 192.168.10.100:25565 | |
| Matrix | 8448 | 192.168.10.100:8448 | Federation |
| WireGuard | 51820/UDP | router | VPN |

## IPv6

Hurricane Electric 6in4 tunnel provides IPv6 connectivity. Configuration details in secrets.
