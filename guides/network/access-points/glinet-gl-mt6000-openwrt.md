---
type: guide
title: 'GL.iNet GL-MT6000: OpenWrt flashing guide'
description: Hardware specs and how to flash vanilla OpenWrt onto a GL.iNet GL-MT6000 (Flint 2) from its OpenWrt-based stock firmware via a normal sysupgrade.
tags: [openwrt, router, glinet, mt6000, hardware, flashing]
status: draft
resource:
created: 2026-09-28T17:04:00Z
updated: 2026-09-28T17:04:00Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:04:00Z
verified: []
stale_after: 2027-09-28T17:04:00Z
sources:
- id: 2026-09-28-openwrt-device-model-docs
  resource: 'private:/sources/network/access-points/2026-09-28-openwrt-device-model-docs.md'
relations: []
superseded_by:
---

# GL.iNet GL-MT6000 ("Flint 2"): OpenWrt flashing guide

Compact travel/home router built on MediaTek's Filogic 830 platform. GL.iNet ships this with their
own OpenWrt-based firmware, which makes migrating to vanilla (upstream) OpenWrt unusually painless
— no exploit or bootloader dance needed, just a normal firmware upload through the stock web UI.

## Hardware specifications

| Property | Value |
|---|---|
| SoC | MediaTek Filogic 830 |
| CPU | 4x ARM Cortex-A53 @ 2.0GHz (aarch64) |
| RAM | 1GB DDR4 |
| Flash | 8GB eMMC |
| Ethernet | 2x 2.5Gbit WAN + 4x 1Gbit LAN (MediaTek 1Gbit switch + 2x Realtek 2.5Gbit PHYs) |
| USB | USB 3.2 |
| WiFi | Dual-band WiFi 6, 4x4 |
| OpenWrt target | `mediatek/filogic` |

## Default network

| | Stock GL.iNet firmware | OpenWrt (post-flash) |
|---|---|---|
| Web UI / router IP | `192.168.8.1` | `192.168.1.1` |  <!-- wiki:allow -->

## Steps: flashing vanilla OpenWrt

GL.iNet's stock firmware is itself OpenWrt-based, so this is a normal sysupgrade — no RCE exploit
required.

### Method 1: web UI (recommended)

1. Download the OpenWrt **sysupgrade** image for `mediatek/filogic`, board `glinet_gl-mt6000`,
   from the [OpenWrt downloads page](https://downloads.openwrt.org/releases/) — always use the
   sysupgrade image, not the factory image, when coming from GL.iNet stock firmware.
2. Log into the stock web UI at `192.168.8.1`.  <!-- wiki:allow -->
3. Go to the firmware Upgrade page, upload the sysupgrade image.
4. Select **"Do not keep configuration"** when prompted.
5. The device reboots into vanilla OpenWrt at `192.168.1.1`.  <!-- wiki:allow -->

### Method 2: U-Boot web recovery (fallback)

If the device is unresponsive or you want a clean recovery path:

1. Hold the **Reset** button while powering the device on to enter the U-Boot web recovery
   interface.
2. Set your PC to a static IP `192.168.1.2`, connect via a LAN port.  <!-- wiki:allow -->
3. Browse to `192.168.1.1` — the U-Boot web interface should load.  <!-- wiki:allow -->
4. Upload the sysupgrade image there.

## Verify

```sh
cat /etc/openwrt_release
ubus call system board
```

## Related

<None yet.>

## Sources

- [OpenWrt Table of Hardware: GL.iNet GL-MT6000](https://openwrt.org/toh/gl.inet/gl-mt6000)
- [OpenWrt firmware selector — GL-MT6000](https://firmware-selector.openwrt.org/?target=mediatek%2Ffilogic&id=glinet_gl-mt6000)
- [Legacy wiki.js: OpenWrt AP hardware device model pages](../../../../../sources/network/access-points/2026-09-28-openwrt-device-model-docs.md)
