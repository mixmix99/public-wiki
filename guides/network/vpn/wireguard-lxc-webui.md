---
type: guide
title: Running WireGuard with WireGuard-UI in a Proxmox LXC container
description: Prepare an unprivileged Proxmox LXC container for WireGuard's TUN device, install WireGuard-UI as a web front-end, and auto-apply config changes via a systemd path unit.
tags: [wireguard, proxmox, lxc, vpn, systemd]
status: draft
resource:
created: 2026-09-28T17:01:19Z
updated: 2026-09-28T17:01:19Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:01:19Z
verified: []
stale_after: 2027-09-28T17:01:19Z
sources:
- id: 2026-09-28-wireguard-lxc-webui-setup
  resource: 'private:/sources/network/vpn/2026-09-28-wireguard-lxc-webui-setup.md'
relations: []
superseded_by:
---

# Running WireGuard with WireGuard-UI in a Proxmox LXC container

Run a [WireGuard](https://www.wireguard.com/) endpoint inside an unprivileged Proxmox LXC
container, managed through the [WireGuard-UI](https://github.com/ngoduykhanh/wireguard-ui) web
front-end instead of hand-editing config files — useful when you want a dedicated, lightweight VPN
container separate from your firewall/router.

## Prerequisites

- A Proxmox VE host.
- Comfortable editing an LXC container's `.conf` file on the Proxmox host directly.

## Steps

### 1. Prepare the Proxmox host

Unprivileged containers need the host's TUN device made accessible to the container's mapped root
UID (100000 by default):

```bash
chown 100000:100000 /dev/net/tun
```

### 2. Create the LXC container

Create it **unprivileged**, with **nesting disabled**. Sizing is flexible — a minimal container
(a few CPU cores, no swap, ~1 GB RAM) is enough for a WireGuard endpoint plus its web UI.

### 3. Expose the TUN device inside the container

Edit the container's config file on the Proxmox host (`/etc/pve/lxc/<container-id>.conf`) and add:

```
lxc.cgroup.devices.allow: c 10:200 rwm
lxc.mount.entry: /dev/net dev/net none bind,create=dir
```

Restart the container for this to take effect.

### 4. Install WireGuard inside the container

Update the container OS, then install WireGuard via a one-shot install script — its interactive
prompts don't matter much, since WireGuard-UI will overwrite the resulting config anyway:

```bash
apt-get update && apt-get upgrade -y && apt-get dist-upgrade && apt-get autoremove -y
apt install wget curl -y
wget git.io/wireguard -O wireguard-install.sh && bash wireguard-install.sh
```

### 5. Install and run WireGuard-UI

Download [WireGuard-UI](https://github.com/ngoduykhanh/wireguard-ui) into `/etc/wireguard/` and run
it as a systemd service bound to port 80:

```bash
#!/bin/bash
# /etc/wireguard/start-wgui.sh
cd /etc/wireguard
./wireguard-ui -bind-address 0.0.0.0:80
```

```ini
# /etc/systemd/system/wgui-web.service
[Unit]
Description=WireGuard UI

[Service]
Type=simple
ExecStart=/etc/wireguard/start-wgui.sh

[Install]
WantedBy=multi-user.target
```

An update script keeps WireGuard-UI itself current by pulling the latest GitHub release:

```bash
#!/bin/bash
# /etc/wireguard/update.sh
VER=$(curl -sI https://github.com/ngoduykhanh/wireguard-ui/releases/latest | grep "location:" | cut -d "/" -f8 | tr -d '\r')
echo "downloading wireguard-ui $VER"
curl -sL "https://github.com/ngoduykhanh/wireguard-ui/releases/download/$VER/wireguard-ui-$VER-linux-amd64.tar.gz" -o wireguard-ui-$VER-linux-amd64.tar.gz
echo -n "extracting "; tar xvf wireguard-ui-$VER-linux-amd64.tar.gz
echo "restarting wgui-web.service"
systemctl restart wgui-web.service
```

```bash
chmod +x /etc/wireguard/start-wgui.sh
chmod +x /etc/wireguard/update.sh
cd /etc/wireguard; ./update.sh
```

### 6. Auto-apply WireGuard-UI's config changes

WireGuard-UI writes its changes to `/etc/wireguard/wg0.conf` but does not restart the WireGuard
interface itself. A systemd path unit watches the file and restarts `wg-quick@wg0` whenever it
changes:

```ini
# /etc/systemd/system/wgui.service
[Unit]
Description=Restart WireGuard
After=network.target

[Service]
Type=oneshot
ExecStart=/bin/systemctl restart wg-quick@wg0.service

[Install]
RequiredBy=wgui.path
```

```ini
# /etc/systemd/system/wgui.path
[Unit]
Description=Watch /etc/wireguard/wg0.conf for changes

[Path]
PathModified=/etc/wireguard/wg0.conf

[Install]
WantedBy=multi-user.target
```

Enable and start everything:

```bash
/etc/wireguard/update.sh
touch /etc/wireguard/wg0.conf
systemctl enable wgui.{path,service} wg-quick@wg0.service wgui-web.service
systemctl start wgui.{path,service}
```

### 7. NAT / forwarding

Add standard iptables rules so tunnel traffic is forwarded and NAT'd out the container's egress
interface (adjust `eth0` and `%i` — WireGuard substitutes `%i` with the interface name — to your
setup):

```
# PostUp
iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

```
# PostDown
iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

## Verify

- WireGuard-UI's web interface loads on port 80 of the container.
- Editing a peer in WireGuard-UI and saving triggers `wgui.path` → `wgui.service`, which restarts
  `wg-quick@wg0` — confirm with `systemctl status wgui.service` and `wg show wg0` showing the new
  peer.
- A WireGuard client can connect and route traffic through the container.

## Troubleshooting

- **TUN device not available inside the container:** re-check the host-side `chown` (step 1) and
  the two `lxc.cgroup.devices.allow`/`lxc.mount.entry` lines in the container config (step 3) — both
  are required for an unprivileged container, and a typo or missing restart after editing the
  `.conf` file is the most common cause.
- **WireGuard-UI changes don't take effect:** confirm `wgui.path` is active
  (`systemctl status wgui.path`) — if the path unit isn't watching, config edits sit in
  `wg0.conf` without ever restarting the WireGuard interface.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: WireGuard in an LXC container with WireGuard-UI](../../../../../sources/network/vpn/2026-09-28-wireguard-lxc-webui-setup.md) — private source (original wiki.js text)
