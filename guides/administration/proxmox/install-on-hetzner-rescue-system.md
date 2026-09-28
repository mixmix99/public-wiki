---
type: guide
title: Installing Proxmox VE on a Hetzner dedicated server via the Rescue System
description: Use Hetzner's Rescue System plus a QEMU-hosted installer to install Proxmox VE on a dedicated root server that has no physical console access.
tags: [proxmox, hetzner, installation, rescue-system, qemu]
status: draft
resource:
created: 2026-09-28T17:03:37Z
updated: 2026-09-28T17:03:37Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:03:37Z
verified: []
stale_after: 2027-09-28T17:03:37Z
sources:
- id: 2026-09-28-install-proxmox-on-hetzner-rescue
  resource: 'private:/sources/administration/proxmox/2026-09-28-install-proxmox-on-hetzner-rescue.md'
relations: []
superseded_by:
---

# Installing Proxmox VE on a Hetzner dedicated server via the Rescue System

A Hetzner **dedicated root server** has no physical/virtual console you can boot an installer ISO
from directly (unlike a local machine or a cloud VM with a virtual KVM). Hetzner's Rescue System —
a temporary Linux environment you boot into over the network — combined with running the Proxmox
VE installer **inside QEMU** and viewing it over VNC gives you a way to run the normal graphical
installer remotely.

> This only applies to a Hetzner dedicated/root server. A Hetzner **Cloud VM** already boots
> normally from an ISO you attach in the Cloud Console and doesn't need this workaround.

## Prerequisites

- A Hetzner Robot account with access to the target server.
- A VNC client to view the installer.

## Steps

### 1. Boot into the Rescue System

1. In Hetzner Robot, select the server → **Rescue** tab → activate the Linux 64-bit rescue variant.
   You're given a one-time root password for SSH.
2. Restart the server (via the OS, or Robot's **Reset** tab if it's unreachable). The Rescue System
   activation is valid for one boot only, and expires if the server isn't rebooted within 60
   minutes — if it is rebooted later, it boots normally instead.
3. SSH in as root with the given password:
   ```bash
   ssh root@<SERVER_IP>
   ```

### 2. Fetch the Proxmox VE ISO

```bash
ISO_VERSION=$(curl -s 'http://download.proxmox.com/iso/' | grep -oP 'proxmox-ve_(\d+.\d+-\d).iso' | sort -V | tail -n1)
ISO_URL="http://download.proxmox.com/iso/$ISO_VERSION"
curl $ISO_URL -o /tmp/proxmox-ve.iso
```

### 3. Capture the server's network configuration

You'll need to re-apply this after installation, since the installer doesn't know Hetzner's
network setup:

```bash
INTERFACE_NAME=$(udevadm info -q property /sys/class/net/eth0 | grep "ID_NET_NAME_PATH=" | cut -d'=' -f2)
IP_CIDR=$(ip addr show eth0 | grep "inet\b" | awk '{print $2}')
GATEWAY=$(ip route | grep default | awk '{print $3}')
IP_ADDRESS=$(echo "$IP_CIDR" | cut -d'/' -f1)
CIDR=$(echo "$IP_CIDR" | cut -d'/' -f2)
```

### 4. Boot the installer inside QEMU

```bash
PRIMARY_DISK=$(lsblk -dn -o NAME,SIZE,TYPE -e 1,7,11,14,15 | sed -n 1p | awk '{print $1}')
SECONDARY_DISK=$(lsblk -dn -o NAME,SIZE,TYPE -e 1,7,11,14,15 | sed -n 2p | awk '{print $1}')

qemu-system-x86_64 -daemonize -enable-kvm -m 10240 \
  -hda /dev/$PRIMARY_DISK \
  -hdb /dev/$SECONDARY_DISK \
  -cdrom /tmp/proxmox-ve.iso -boot d -vnc :0,password -monitor telnet:127.0.0.1:4444,server,nowait

echo "change vnc password <VNC_PASSWORD>" | nc -q 1 127.0.0.1 4444
```

The server's real disks are passed straight into QEMU as raw block devices, so the installer
writes directly to them. Connect a VNC viewer to `<SERVER_IP>:5900` with the password you set, and
run through the graphical Proxmox VE installer as normal.

### 5. Stop QEMU after the manual install finishes

```bash
printf "quit\n" | nc 127.0.0.1 4444
```

### 6. Restart QEMU without the install CD-ROM

```bash
qemu-system-x86_64 -daemonize -enable-kvm -m 10240 \
  -hda /dev/$PRIMARY_DISK \
  -hdb /dev/$SECONDARY_DISK \
  -vnc :0,password -monitor telnet:127.0.0.1:4444,server,nowait \
  -net user,hostfwd=tcp::2222-:22 -net nic

echo "change vnc password <VNC_PASSWORD>" | nc -q 1 127.0.0.1 4444
```

A user-mode NIC with a port-forward (`2222` → guest port `22`) gives you SSH access into the freshly
installed system for the next step, without needing to configure real networking inside QEMU.

### 7. Push the captured network config into the new system

```bash
cat > /tmp/proxmox_network_config << EOF
auto lo
iface lo inet loopback

iface $INTERFACE_NAME inet manual

auto vmbr0
iface vmbr0 inet static
  address $IP_ADDRESS/$CIDR
  gateway $GATEWAY
  bridge_ports $INTERFACE_NAME
  bridge_stp off
  bridge_fd 0
EOF

sshpass -p "<ROOT_PASSWORD>" scp -o StrictHostKeyChecking=no -P 2222 /tmp/proxmox_network_config root@localhost:/etc/network/interfaces
sshpass -p "<ROOT_PASSWORD>" ssh -o StrictHostKeyChecking=no -p 2222 root@localhost "sed -i 's/nameserver.*/nameserver 1.1.1.1/' /etc/resolv.conf"
```

### 8. Shut down QEMU and reboot into the real install

```bash
printf "system_powerdown\n" | nc 127.0.0.1 4444
shutdown -r now
```

The Rescue System itself reboots, and — since the Rescue activation only applies to one boot —
the server comes up from its actual disks into the newly installed Proxmox VE.

## Verify

After the reboot, the Proxmox VE web UI should be reachable at `https://<YourIPAddress>:8006`.

## Related

- [A private NAT network for internal VMs on a Hetzner root server](hetzner-private-nat-network.md)
  — natural next step once Proxmox is installed, if internal-only VMs are needed.

## Sources

- [Legacy wiki.js: Install Proxmox VE on Hetzner via Rescue System (private)](../../../../../sources/administration/proxmox/2026-09-28-install-proxmox-on-hetzner-rescue.md)
