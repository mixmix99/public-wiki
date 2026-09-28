---
type: concept
title: OpenMediaVault as a Debian-based NAS distribution
description: What OpenMediaVault is, its benefits over building NAS storage management by hand, and its installer/default-login quirks.
tags: [nas, omv, debian, storage]
status: draft
resource:
created: 2026-09-28T17:04:56Z
updated: 2026-09-28T17:04:56Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:04:56Z
verified: []
stale_after: 2028-09-27T17:04:56Z
sources:
- id: 2026-09-28-openmediavault-overview
  resource: 'private:/sources/administration/linux/2026-09-28-openmediavault-overview.md'
relations: []
superseded_by:
---

# OpenMediaVault as a Debian-based NAS distribution

OpenMediaVault (OMV) is a Debian-based NAS/RAID storage management distribution: a regular Debian
system with a web UI and plugin ecosystem layered on top for disk, share and RAID management,
rather than a from-scratch appliance OS.

## How it works

Install it like any Debian-derived distribution (there's a guided installer); afterward,
day-to-day administration happens through its web UI rather than the shell. Storage features
(Btrfs support, RAID 5/6/10) and everything else are exposed as UI screens over standard Linux
tooling (mdadm, Btrfs, Samba, NFS, etc.) — being Debian underneath means anything not covered by
the UI is still reachable directly from a shell.

## When to use it / trade-offs

- **Benefits:** being Debian-based means broad hardware/driver support and a huge package
  ecosystem underneath the UI; it's resource-efficient compared to heavier NAS appliance OSes; it
  natively supports RAID 5/6/10 and Btrfs alongside the more common ext4/XFS/ZFS options other NAS
  distributions favor.
- **Trade-off vs. a ZFS-first NAS OS (e.g. TrueNAS):** OMV's storage story centers on
  mdadm/Btrfs rather than ZFS, so pick based on which storage stack (and its snapshot/scrub/
  checksum tooling) you actually want to standardize on, not just "which NAS distro."
- **Extensibility:** the OMV-Extras plugin repository extends the base install significantly;
  install it via a one-line script fetched directly from its GitHub repo (`wget -O - <url> | bash`)
  — the usual caution around piping a remote script straight into a shell applies.

## Pitfalls

- **Installer can appear to hang after network card detection.** If it does, cancel with Ctrl+C;
  on the *next* detection prompt, press Enter to cancel again and the installation continues
  normally. This looks like a crash the first time you hit it.
- **The web UI login is not the Debian install-time user.** The installer's user account is for
  the underlying OS; the web UI has its own separate account, defaulting to `admin` /
  `openmediavault` <!-- wiki:allow --> (documented vendor default — change it on first login like
  any other appliance default credential).

## Related

<None yet.>

## Sources

- [Legacy wiki.js: OpenMediaVault overview](../../../../../sources/administration/linux/2026-09-28-openmediavault-overview.md) — private source
