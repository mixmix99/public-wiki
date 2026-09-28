---
type: guide
title: Exposing AMD Ryzen sensors on Proxmox VE / Debian
description: Install lm-sensors and run sensors-detect to expose AMD Ryzen temperature/fan sensors on a Proxmox VE or Debian host.
tags: [proxmox, debian, linux, sensors, amd, ryzen, hardware]
status: draft
resource:
created: 2026-09-28T18:54:10Z
updated: 2026-09-28T18:54:10Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:54:10Z
verified: []
stale_after: 2027-09-28T18:54:10Z
sources:
- id: 2026-09-28-amd-ryzen-sensors
  resource: 'private:/sources/administration/proxmox/2026-09-28-amd-ryzen-sensors.md'
relations: []
superseded_by:
---

# Exposing AMD Ryzen sensors on Proxmox VE / Debian

Debian (and Proxmox VE, which is Debian-based) does not read out AMD Ryzen hardware sensors
(temperatures, fan speeds, voltages) by default. Installing `lm-sensors` and running its detection
wizard exposes them.

## Prerequisites

- Root/sudo access on the Proxmox VE or Debian host.

## Steps

### 1. Install lm-sensors and build dependencies

```bash
apt install lm-sensors bison flex librrd-dev
```

### 2. Run sensor detection

```bash
sensors-detect
```

Answer **yes** to every prompt — this loads all the kernel modules the wizard identifies as
applicable for the hardware.

## Verify

```bash
sensors
```

Should now list AMD Ryzen temperature/voltage/fan sensors.

## Related

- [Reducing swap usage on a hypervisor host to protect SSD lifespan](../proxmox/tuning-swappiness.md) — another small generic Proxmox VE host tweak.
- [Removing the Proxmox VE subscription notice](../proxmox/removing-proxmox-subscription-notice.md) — another small generic Proxmox VE host tweak.

## Sources

- [Legacy wiki.js (de, translated): Show AMD Ryzen sensors on Proxmox/Debian](../../../../../sources/administration/proxmox/2026-09-28-amd-ryzen-sensors.md) — private source; German-only wiki.js page, no English original existed
