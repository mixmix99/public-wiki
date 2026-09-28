---
type: guide
title: Configuring DNS resolution with systemd-resolved
description: Set up systemd-resolved for asynchronous, cached DNS resolution on a Linux host, including the symlink needed to actually use it and a Proxmox LXC gotcha.
tags:
- linux
- dns
- systemd
- resolved
status: draft
resource:
created: 2026-09-28T17:06:07Z
updated: 2026-09-28T18:56:03Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:56:03Z
verified: []
stale_after: 2027-09-28T17:06:07Z
sources:
- id: 2026-09-28-storage-and-dns-tools
  resource: private:/sources/administration/linux/2026-09-28-storage-and-dns-tools.md
- id: 2026-09-28-static-resolv-conf
  resource: private:/sources/administration/linux/2026-09-28-static-resolv-conf.md
relations: []
superseded_by:
---

# Configuring DNS resolution with systemd-resolved

`systemd-resolved` is a systemd service that resolves DNS queries asynchronously and in parallel,
with its own cache — it can be used alongside, or instead of, traditional resolution via
`/etc/hosts` and a static `/etc/resolv.conf`.

## Prerequisites

- A systemd-based Linux distribution.
- Root/sudo access.

## Steps

### 1. Configure resolved

Edit `/etc/systemd/resolved.conf`, in the `[Resolve]` section:

```
[Resolve]
DNS=<dns-server-ip>
Domains=<search-domain>
```

### 2. Enable and start the service

```bash
systemctl start systemd-resolved
systemctl enable systemd-resolved
systemctl status systemd-resolved
```

### 3. Point /etc/resolv.conf at resolved

Configuration alone isn't enough — resolution still needs to actually be routed through resolved:

```bash
rm /etc/resolv.conf
ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

## Verify

```bash
nslookup <some-domain>
```

Expect a response via the local stub resolver (`127.0.0.53#53`). Check live statistics:

```bash
resolvectl statistics
watch -n 1 resolvectl statistics
```

## Troubleshooting

- **DNS still resolves the "old" way after configuring resolved.conf:** the symlink step (step 3)
  is required — editing `resolved.conf` alone does not repoint `/etc/resolv.conf`.
- **Running inside a Proxmox LXC container and `/etc/resolv.conf` keeps reverting:** Proxmox
  overwrites `/etc/resolv.conf` on container start by default. Create an empty marker file to stop
  it:

  ```bash
  touch /etc/.pve-ignore.resolv.conf
  ```

  After that, the steps above apply normally inside the container.
- **Need to change the DNS server later:** it must be changed in `/etc/systemd/resolved.conf`
  itself (and the service restarted), not via `/etc/resolv.conf`, since that file is now just a
  symlink to the resolved stub.

## Alternative: static resolv.conf

`systemd-resolved` isn't the only way to pin a DNS server on Ubuntu/Debian. A simpler, more
"static" alternative: delete the `/etc/resolv.conf` symlink entirely and replace it with a plain,
non-symlinked file that nothing manages or regenerates automatically.

```bash
sudo rm /etc/resolv.conf
echo "nameserver <dns-server-ip>
options edns0 trust-ad
search <search-domain>" | sudo tee /etc/resolv.conf > /dev/null
sudo chmod 644 /etc/resolv.conf
```

Verify with:

```bash
cat /etc/resolv.conf
```

**Trade-offs versus `systemd-resolved`:**

- No local caching or asynchronous/parallel resolution — every lookup goes straight to the
  configured server(s).
- It permanently overrides whatever DHCP or other network management would otherwise set — useful
  when that's exactly the point (a server that must never silently pick up a different resolver),
  but it means the file has to be revisited manually if the network's DNS setup changes, and
  anything that expects to manage `/etc/resolv.conf` (NetworkManager, `systemd-resolved` itself,
  DHCP client hooks) will either fight with it or silently have no effect.
- No `resolvectl`-style per-interface visibility or statistics — just the one flat file.

Use this when a host needs a DNS setting that absolutely will not change without a manual edit
(e.g. deliberately bypassing DHCP-provided DNS); use `systemd-resolved` (above) for the caching,
async resolution and per-interface flexibility instead.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: DNS Resolution with resolved](../../../../../sources/administration/linux/2026-09-28-storage-and-dns-tools.md) — private source
- [Legacy wiki.js (de, translated): Ubuntu static resolv.conf](../../../../../sources/administration/linux/2026-09-28-static-resolv-conf.md) — private source; German-only wiki.js page, no English original existed
