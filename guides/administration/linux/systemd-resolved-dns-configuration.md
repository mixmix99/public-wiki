---
type: guide
title: Configuring DNS resolution with systemd-resolved
description: Set up systemd-resolved for asynchronous, cached DNS resolution on a Linux host, including the symlink needed to actually use it and a Proxmox LXC gotcha.
tags: [linux, dns, systemd, resolved]
status: draft
resource:
created: 2026-09-28T17:06:07Z
updated: 2026-09-28T17:06:07Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:06:07Z
verified: []
stale_after: 2027-09-28T17:06:07Z
sources:
- id: 2026-09-28-storage-and-dns-tools
  resource: 'private:/sources/administration/linux/2026-09-28-storage-and-dns-tools.md'
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

## Related

<None yet.>

## Sources

- [Legacy wiki.js: DNS Resolution with resolved](../../../../../sources/administration/linux/2026-09-28-storage-and-dns-tools.md) — private source
