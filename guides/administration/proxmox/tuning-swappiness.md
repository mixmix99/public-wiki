---
type: guide
title: Reducing swap usage on a hypervisor host to protect SSD lifespan
description: Lower Linux swappiness on a Proxmox VE (or any Linux) host to reduce swap-to-SSD writes, with the commands to check, change and verify it.
tags:
- proxmox
- linux
- swap
- ssd
- performance
status: draft
resource:
created: 2026-09-28T17:03:37Z
updated: 2026-09-28T18:55:38Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:55:38Z
verified: []
stale_after: 2027-09-28T17:03:37Z
sources:
- id: 2026-09-28-proxmox-helper-tweaks
  resource: private:/sources/administration/proxmox/2026-09-28-proxmox-helper-tweaks.md
relations: []
superseded_by:
---

# Reducing swap usage on a hypervisor host to protect SSD lifespan

The Linux kernel's default `vm.swappiness` (60 on most distributions) makes the kernel start
swapping out fairly aggressively, well before physical memory is actually exhausted. On a
hypervisor host with plenty of RAM, this can mean noticeable and unnecessary swap-to-SSD write
traffic — which shortens SSD lifespan over time for no real benefit.

## How it works

`vm.swappiness` is a 0–100 kernel tunable controlling how aggressively the kernel prefers to swap
out anonymous memory pages versus reclaiming page cache:

- `vm.swappiness=0` — swap only when physical memory is nearly exhausted.
- `vm.swappiness=10` — swap once free memory drops to roughly 10%.
- `vm.swappiness=60` (typical default) — swaps considerably earlier and more readily.

## Steps

### 1. Check the current value

```bash
cat /proc/sys/vm/swappiness
```

### 2. Set a new value

```bash
sysctl vm.swappiness=0
```

Use `0` to swap only under real memory pressure, or a low value like `10` for a small safety
margin. Make it persistent across reboots by adding it to `/etc/sysctl.conf` or a file under
`/etc/sysctl.d/`:

```
vm.swappiness=0
```

### 3. Clear out existing swap usage (optional)

Applying the new `swappiness` value doesn't undo memory already swapped out. To force everything
back into RAM immediately:

```bash
swapoff -a   # can take several minutes on hosts with a lot of swapped data
swapon -a
```

![Setting swappiness to 0 and cycling swap off/on in the Proxmox web
shell](tuning-swappiness/proxmox-shell-console.png)

## Verify

```bash
cat /proc/sys/vm/swappiness
```

## Troubleshooting

- **`swapoff -a` seems to hang**: it can legitimately take on the order of 10–15 minutes on a host
  with a large amount of data currently in swap, since every swapped page has to be paged back into
  RAM before the swap area can be released — this is expected, not a fault.

## Related

- [Removing the Proxmox VE subscription notice](removing-proxmox-subscription-notice.md) — another
  small generic Proxmox VE host tweak from the same source.
- [Exposing AMD Ryzen sensors on Proxmox VE / Debian](amd-ryzen-sensors.md) — another small generic
  Proxmox VE host tweak.

## Sources

- [Legacy wiki.js: Proxmox helper tweaks (private)](../../../../../sources/administration/proxmox/2026-09-28-proxmox-helper-tweaks.md)
