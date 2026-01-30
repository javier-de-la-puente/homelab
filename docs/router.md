# OpenWRT Router Management

## Hardware

- **Device:** Raspberry Pi 4 (4GB)
- **eth0:** Built-in Gigabit Ethernet - LAN trunk to switches
- **eth1:** USB 3.0 Ethernet adapter (Realtek RTL8153) - WAN via PPPoE (VLAN 20)
- **wlan0:** Built-in WiFi (5GHz) - Management backup access

### USB Ethernet Adapter

The WAN interface uses a Realtek RTL8153 USB Ethernet adapter. This requires the `kmod-usb-net-rtl8152` kernel module which is **not included** in the default OpenWRT image.

**ISP MAC requirement:** The adapter MAC must be spoofed to `68:77:da:78:1c:47` (configured in Ansible).

## Initial Setup

### Prerequisites

1. **Set WiFi country code** (required for Pi WiFi to work in OpenWRT):
   ```bash
   # Flash Raspberry Pi OS Lite to SD card first
   # Boot, then:
   sudo raspi-config
   # Navigate: Localisation Options → WLAN Country → ES (Spain)
   sudo shutdown now
   ```

2. **Update firmware** (recommended while in Pi OS):
   ```bash
   sudo apt update && sudo apt full-upgrade -y
   sudo rpi-eeprom-update -a
   sudo reboot
   ```

3. **Flash OpenWRT** to SD card (overwrites Pi OS)

4. **Download USB Ethernet driver packages** (required for WAN - do this on a machine with internet):

   The RTL8153 adapter requires kernel modules not included in base OpenWRT. Download these packages for offline installation.

   ```bash
   # Create directory for packages
   mkdir -p openwrt-packages && cd openwrt-packages

   # Kernel modules (from kmods directory - hash is specific to 24.10.4)
   KMODS_URL="https://downloads.openwrt.org/releases/24.10.4/targets/bcm27xx/bcm2711/kmods/6.6.110-1-5642ee3ae5a6da3ce336f51cf968083c"

   wget "${KMODS_URL}/kmod-mii_6.6.110-r1_aarch64_cortex-a72.ipk"
   wget "${KMODS_URL}/kmod-crypto-hash_6.6.110-r1_aarch64_cortex-a72.ipk"
   wget "${KMODS_URL}/kmod-crypto-sha256_6.6.110-r1_aarch64_cortex-a72.ipk"
   wget "${KMODS_URL}/kmod-usb-net-cdc-ncm_6.6.110-r1_aarch64_cortex-a72.ipk"
   wget "${KMODS_URL}/kmod-usb-net-rtl8152_6.6.110-r1_aarch64_cortex-a72.ipk"

   # Firmware (from base packages)
   BASE_URL="https://downloads.openwrt.org/releases/24.10.4/packages/aarch64_cortex-a72/base"

   wget "${BASE_URL}/r8152-firmware_20241110-r2_aarch64_cortex-a72.ipk"
   ```

   **Note:** The kernel hash in the URL (`6.6.110-1-5642ee3ae5a6da3ce336f51cf968083c`) is specific to OpenWRT 24.10.4 for RPi4 (bcm2711). For other versions, browse https://downloads.openwrt.org/releases/ to find the correct kmods path.

   Copy packages to a USB drive or the SD card's boot partition.

### Stage 1: Bootstrap

Connect Pi eth0 directly to laptop. Configure laptop with static IP `192.168.1.2/24`.

```bash
# Install SSH key for passwordless access (required for Ansible)
ssh root@192.168.1.1 "tee -a /etc/dropbear/authorized_keys" < ~/.ssh/id_ed25519.pub
```

**Install USB Ethernet driver and Python:**

**Alternative: Share laptop internet with Pi**

Instead of downloading packages manually, share your laptop's internet connection:

```bash
# On laptop - find internet interface (e.g., wlan0)
ip route | grep default

# Enable NAT (replace wlan0 with your interface)
sudo sysctl -w net.ipv4.ip_forward=1
sudo iptables -t nat -A POSTROUTING -o wlan0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

```bash
# On router - add route and DNS
ip route add default via 192.168.1.2
echo "nameserver 1.1.1.1" > /etc/resolv.conf

# Fix date (required for HTTPS - adjust to current time)
date -s "2025-01-29 20:30:00"

# Install USB driver, Python, and SFTP server (for Ansible)
opkg update
opkg install kmod-usb-net-rtl8152 python3 openssh-sftp-server
```

**Alternative: HTTP server on laptop**

Fresh OpenWRT doesn't have scp. Use a simple HTTP server on your laptop to serve the packages:

```bash
# On laptop - start HTTP server in packages directory
cd openwrt-packages
python3 -m http.server 8000
```

Then on the router, download and install:

```bash
ssh root@192.168.1.1

# Download packages from laptop (laptop is at 192.168.1.2)
cd /tmp
wget http://192.168.1.2:8000/kmod-mii_6.6.110-r1_aarch64_cortex-a72.ipk
wget http://192.168.1.2:8000/kmod-crypto-hash_6.6.110-r1_aarch64_cortex-a72.ipk
wget http://192.168.1.2:8000/kmod-crypto-sha256_6.6.110-r1_aarch64_cortex-a72.ipk
wget http://192.168.1.2:8000/r8152-firmware_20241110-r2_aarch64_cortex-a72.ipk
wget http://192.168.1.2:8000/kmod-usb-net-cdc-ncm_6.6.110-r1_aarch64_cortex-a72.ipk
wget http://192.168.1.2:8000/kmod-usb-net-rtl8152_6.6.110-r1_aarch64_cortex-a72.ipk

# Install in dependency order
opkg install /tmp/kmod-mii_*.ipk
opkg install /tmp/kmod-crypto-hash_*.ipk
opkg install /tmp/kmod-crypto-sha256_*.ipk
opkg install /tmp/r8152-firmware_*.ipk
opkg install /tmp/kmod-usb-net-cdc-ncm_*.ipk
opkg install /tmp/kmod-usb-net-rtl8152_*.ipk

# Verify eth1 appears (plug in USB adapter if not already)
ip link show eth1

exit
```

**Run Ansible bootstrap** (from laptop):
```bash
cd ansible
ansible-playbook playbooks/openwrt-bootstrap.yml -i inventory/bootstrap/openwrt.yml
```

After bootstrap:
- Router IP changes to **10.50.1.1**
- Management WiFi SSID: **homelab-mgmt**
- PPPoE WAN should connect automatically

### Stage 2: Full Configuration

Connect to router via WiFi or switch, then:

```bash
cd ansible
ansible-playbook playbooks/openwrt-full.yml -i inventory/hosts.yml -l router
```

This configures:
- All VLANs (1, 10, 11, 20, 30, 40, 50, 60)
- Firewall zones and inter-VLAN policies
- DHCP pools and static reservations
- WireGuard VPN
- IPv6 tunnel
- Port forwards

## Common Tasks

### Adding a WireGuard Peer

1. Generate keys on the client:
   ```bash
   wg genkey | tee privatekey | wg pubkey > publickey
   ```

2. Edit secrets file:
   ```bash
   sops ansible/group_vars/openwrt/secrets.sops.yml
   ```
   Add new peer under `wireguard.peers`:
   ```yaml
   - name: newpeer
     public_key: "PUBLIC_KEY_HERE"
     allowed_ips: "10.50.50.X/32"
   ```

3. Re-run Ansible:
   ```bash
   ansible-playbook playbooks/openwrt-full.yml -l router
   ```

4. Configure client with server details:
   ```ini
   [Interface]
   PrivateKey = CLIENT_PRIVATE_KEY
   Address = 10.50.50.X/24
   DNS = 10.50.1.1

   [Peer]
   PublicKey = SERVER_PUBLIC_KEY
   Endpoint = YOUR_PUBLIC_IP:51820
   AllowedIPs = 10.50.0.0/16, 192.168.10.0/24, 192.168.11.0/24
   PersistentKeepalive = 25
   ```

### Adding a Port Forward

Edit `ansible/group_vars/openwrt/firewall.yml`:

```yaml
port_forwards:
  # ... existing forwards ...
  - name: MyService
    src: wan
    proto: tcp
    src_dport: 8080
    dest: lan  # or legacy
    dest_ip: "10.50.60.100"
    dest_port: 8080
```

Re-run Ansible:
```bash
ansible-playbook playbooks/openwrt-full.yml -l router
```

### Adding a DHCP Reservation

Edit `ansible/group_vars/openwrt/dhcp.yml`:

```yaml
static_hosts:
  kubernetes:  # VLAN name
    - name: newhost
      mac: "AA:BB:CC:DD:EE:FF"
      ip: "10.50.60.50"
```

Re-run Ansible:
```bash
ansible-playbook playbooks/openwrt-full.yml -l router
```

## Troubleshooting

### Can't Connect to Router

1. **Try management WiFi:** Connect to `homelab-mgmt` SSID
2. **Direct connection:** Connect laptop to Pi eth0 with static IP in 10.50.1.0/24
3. **Serial console:** Connect USB-serial to Pi GPIO pins 8 (TX) and 10 (RX)

### WAN Not Connecting

**First, check if USB adapter is recognized:**
```bash
ip link show eth1
# If missing, check kernel module:
lsmod | grep r8152
dmesg | grep -i rtl
```

If eth1 is missing, reinstall the kernel module packages (see Stage 1).

**Check PPPoE status:**
```bash
ssh root@10.50.1.1
logread | grep pppoe
ifstatus wan
```

**Verify MAC address is set correctly:**
```bash
ip link show eth1.20
# Should show: link/ether 68:77:da:78:1c:47
```

Verify credentials in `ansible/group_vars/openwrt/secrets.sops.yml`.

### VLAN Not Working

Verify VLAN interface:
```bash
ip addr show eth0.60  # Replace 60 with VLAN ID
```

Check switch port PVID configuration matches expected VLAN.

### WireGuard Issues

Check interface status:
```bash
wg show wg0
```

Verify firewall allows UDP 51820:
```bash
iptables -L -n | grep 51820
```

## Backup and Recovery

### Backup Current Config

```bash
ssh root@10.50.1.1 "sysupgrade -b -" > openwrt-backup-$(date +%Y%m%d).tar.gz
```

### Restore from Backup

```bash
scp openwrt-backup.tar.gz root@10.50.1.1:/tmp/
ssh root@10.50.1.1 "sysupgrade -r /tmp/openwrt-backup.tar.gz"
```

### Full Reinstall

If all else fails:
1. Flash fresh OpenWRT to SD card
2. Re-run Stage 1 (bootstrap) and Stage 2 (full) playbooks
3. All configuration is in Ansible - nothing to restore manually
