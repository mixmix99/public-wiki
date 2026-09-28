---
type: guide
title: A private NAT network for internal VMs on a Hetzner root server
description: Give internal-only Proxmox VMs outbound internet access via a NAT bridge on a Hetzner dedicated server, without exposing them on the public interface.
tags: [proxmox, hetzner, nat, networking, iptables]
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
- id: 2026-09-28-hetzner-private-nat-network
  resource: 'private:/sources/administration/proxmox/2026-09-28-hetzner-private-nat-network.md'
relations: []
superseded_by:
---

# A private NAT network for internal VMs on a Hetzner root server

On a Hetzner dedicated (root) server, every VM you create is, by default, one bridge away from a
public IP. If a VM only needs outbound internet access (package updates, pulling images) and no
inbound public reachability, putting it on a separate **private NAT bridge** instead of the public
bridge keeps it off the public internet entirely while it still gets outbound connectivity via
masquerading through the host.

## How it works

- `vmbr0` = **public** bridge, attached to the Hetzner uplink, carrying the server's public
  address block.
- `vmbr2` = **private** bridge (e.g. `198.51.100.0/24`), no physical port, with NAT (masquerade) out
  via `vmbr0`.
- VMs on `vmbr2` can talk to each other and reach the internet outbound; only VMs on `vmbr0` are
  publicly reachable.

Only the `MASQUERADE` rules are strictly required, assuming the host's `FORWARD` policy is
`ACCEPT` (Proxmox's default) and IPv4 forwarding is enabled.

## Prerequisites

- IPv4 forwarding enabled at the kernel level:
  ```bash
  grep -Rqs '^\s*net\.ipv4\.ip_forward\s*=\s*1\s*$' /etc/sysctl.d/ /etc/sysctl.conf || \
    printf 'net.ipv4.ip_forward=1\n' >/etc/sysctl.d/99-forwarding.conf
  sysctl --system
  ```
  Verify: `sysctl net.ipv4.ip_forward` → `net.ipv4.ip_forward = 1`.
- `FORWARD` chain policy is `ACCEPT`:
  ```bash
  iptables -S FORWARD
  ```
  If it's `DROP` instead, see the fallback rules at the end of this guide.

## Steps

### 1. Configure the private NAT bridge

Add this stanza to `/etc/network/interfaces`:

```bash
auto vmbr2
iface vmbr2 inet static
        address 198.51.100.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up   iptables -t nat -A POSTROUTING -s '198.51.100.0/24' -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '198.51.100.0/24' -o vmbr0 -j MASQUERADE
```

Tying the `MASQUERADE` rule to the interface's `post-up`/`post-down` hooks means it's applied and
removed automatically with the bridge's own lifecycle — no separate firewall service to manage.

Apply and check:

```bash
ifreload -a   # or: ifdown vmbr2 && ifup vmbr2
ip -4 addr show vmbr2
iptables -t nat -S | grep POSTROUTING
```

### 2. Attach VMs to the right bridge

- **Internal-only VM**: NIC on `vmbr2`, static IP in `198.51.100.x/24` (or DHCP, see step 3),
  gateway `198.51.100.1`, DNS `1.1.1.1` or your own resolver.
- **Publicly reachable VM**: NIC on `vmbr0`, configured with a public address from your Hetzner
  block.

### 3. Optional: DHCP for the private network

A lightweight `dnsmasq` instance is enough for a handful of internal VMs:

```bash
apt update && apt install -y dnsmasq
```

`/etc/dnsmasq.d/vmbr2.conf`:

```ini
interface=vmbr2
bind-interfaces

dhcp-range=198.51.100.100,198.51.100.200,12h
dhcp-option=3,198.51.100.1           # gateway
dhcp-option=6,1.1.1.1,8.8.8.8      # DNS
```

```bash
systemctl enable --now dnsmasq
```

If you later add a dedicated firewall VM (OPNsense/pfSense) to this network, run DHCP inside that
VM instead and disable `dnsmasq` on the host.

### 4. Proxmox firewall

If the built-in Proxmox firewall is enabled at Datacenter/Node level, make sure it doesn't block
forwarding from `vmbr2` to `vmbr0`. For quick triage:

```bash
pve-firewall stop
# test a VM on vmbr2
pve-firewall start
```

## Verify

From the host:

```bash
ip r
iptables -t nat -S | grep MASQUERADE
```

From a VM on `vmbr2`:

```bash
ip a
ip r
ping -c2 198.51.100.1
ping -c2 1.1.1.1
curl -I https://example.com
```

## Troubleshooting

- `net.ipv4.ip_forward` is `0` — internal VMs get no outbound connectivity at all; re-check step 1.
- No `MASQUERADE` rule under `iptables -t nat -S | grep '198.51.100.0/24'` — the bridge's `post-up`
  hook didn't run; re-apply with `ifreload -a`.
- Wrong gateway/DNS on the VM, or Proxmox firewall blocking `FORWARD`.
- Large downloads stall: add an MSS clamp for the forwarded path:
  ```bash
  iptables -t mangle -A FORWARD -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
  ```

### If `FORWARD` policy is `DROP`

Add explicit forward-allow rules, ideally tied to the bridge's own `post-up`/`pre-down` lifecycle
so they don't need a separate service:

```bash
post-up   iptables -C FORWARD -i vmbr2 -o vmbr0 -s 198.51.100.0/24 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT || iptables -A FORWARD -i vmbr2 -o vmbr0 -s 198.51.100.0/24 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
pre-down  iptables -D FORWARD -i vmbr2 -o vmbr0 -s 198.51.100.0/24 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT

post-up   iptables -C FORWARD -i vmbr0 -o vmbr2 -d 198.51.100.0/24 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT || iptables -A FORWARD -i vmbr0 -o vmbr2 -d 198.51.100.0/24 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
pre-down  iptables -D FORWARD -i vmbr0 -o vmbr2 -d 198.51.100.0/24 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Exposing one internal VM's service (port forwarding)

To forward a single public port to a VM on the private network (e.g. host port `54321` → a VM's
SSH on port `22` at `198.51.100.50`):

```bash
# One-time / runtime DNAT rule
iptables -t nat -A PREROUTING -i vmbr0 -p tcp --dport 54321 -j DNAT --to-destination 198.51.100.50:22

# If FORWARD policy is DROP, also allow the forward path explicitly:
iptables -A FORWARD -i vmbr0 -o vmbr2 -p tcp -d 198.51.100.50 --dport 22 -m conntrack --ctstate NEW,ESTABLISHED,RELATED -j ACCEPT
iptables -A FORWARD -i vmbr2 -o vmbr0 -p tcp -s 198.51.100.50 --sport 22 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

Make it persistent by adding equivalent `post-up`/`pre-down` lines (with a `-C ... || -A ...`
idempotency guard) to the `vmbr2` stanza, the same pattern used for the `MASQUERADE` rule above.
Remember to also allow the port through any upstream firewall (e.g. the Hetzner firewall) for
inbound traffic to reach the host at all.

## Related

- [Installing Proxmox VE on a Hetzner dedicated server via the Rescue
  System](install-on-hetzner-rescue-system.md) — the prerequisite install this network setup
  typically follows.

## Sources

- [Legacy wiki.js: Proxmox private NAT network on a Hetzner root server (private)](../../../../../sources/administration/proxmox/2026-09-28-hetzner-private-nat-network.md)
