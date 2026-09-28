---
type: guide
title: 'Cudy AP3000 Outdoor v1: OpenWrt flashing guide'
description: Hardware specs and how to flash OpenWrt onto a Cudy AP3000 Outdoor v1 access point via Cudy's transition firmware, plus a LuCI wireless-status display quirk.
tags: [openwrt, router, cudy, ap3000, outdoor, hardware, flashing]
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

# Cudy AP3000 Outdoor v1: OpenWrt flashing guide

Outdoor-rated (weatherproof enclosure) sibling of the
[Cudy AP3000 v1](cudy-ap3000-v1-openwrt.md), same MediaTek MT7981 filogic platform. Same family,
different board ID and firmware image — **the two are not interchangeable**.

## Hardware specifications

| Property | Value |
|---|---|
| SoC | MediaTek MT7981 (filogic) |
| CPU | 2x ARM Cortex-A53 (aarch64), integrated 2.4GHz + 5GHz radios |
| WiFi | 2.4GHz 802.11n (HT20) + 5GHz 802.11ax (HE80) |
| Enclosure | Outdoor-rated, weatherproof |
| Board ID | `cudy,ap3000outdoor-v1` |
| Firmware filename | `cudy_ap3000outdoor-v1-squashfs-sysupgrade.bin` |
| OpenWrt target | `mediatek/filogic` |

> **Firmware mix-up warning:** the similarly-named `cudy_ap3000-v1` (indoor) image exists and will
> be **rejected by sysupgrade** as the wrong device if you accidentally grab it — always
> double-check the filename says `outdoor` before flashing.

> Newer units manufactured from November 2025 onward (serial numbers starting `2543` or later) use
> a different NAND flash chip (F50L1G41LC). Flashing an older intermediate/OpenWrt image onto these
> can prevent the device from booting — check the
> [OpenWrt forum thread](https://forum.openwrt.org/t/flashing-cudy-ap3000-outdoor-v1/235051) for
> the current guidance on your unit's manufacture date before flashing.

## Default network (stock Cudy firmware)

- Web UI / router IP: `192.168.10.254`  <!-- wiki:allow -->

## Steps: flashing OpenWrt

Same two-stage process as the indoor AP3000 — a Cudy-signed transition image first, then the
official OpenWrt sysupgrade.

### 1. Get the Cudy-signed transition image

Check Cudy's support channel /
[cudytech.com](https://www.cudy.com/en-us/blogs/faq/openwrt-software-download) for the current
transition firmware link for this specific model (`AP3000 Outdoor`, not the indoor `AP3000`).

### 2. Flash the transition image via the stock web UI

1. Log into the stock web UI at `192.168.10.254`.  <!-- wiki:allow -->
2. Upload the transition firmware through the normal firmware-update page.
3. Wait for reboot — device now runs an OpenWrt-based intermediate firmware.

### 3. Flash the official OpenWrt sysupgrade

Download `cudy_ap3000outdoor-v1-squashfs-sysupgrade.bin` for the version you want from the
[OpenWrt downloads page](https://downloads.openwrt.org/releases/), verify the checksum, then flash
via `sysupgrade` (SSH/CLI) or LuCI.

## Known quirk: LuCI wireless status display bug

LuCI's wireless status page may incorrectly show both radios as "Wireless is disabled" with `?` for
channel/bitrate. This is a **cosmetic bug, not a real outage**:

- Root cause: `iwinfo`/`ubus call iwinfo info` reports the hardware as generic `mac80211`
  (unrecognized device-name string for this newer MT7981/mt76 driver) rather than a proper chip
  name.
- LuCI's status JS renders "disabled" whenever it can't fully identify the hardware this way.
- **Verify actual status** with `ubus call network.wireless status` (`up: true`, `disabled: false`)
  and check `hostapd` logs for real client associations — no workaround needed, it's purely a
  display issue.

## Verify

```sh
cat /etc/openwrt_release
ubus call network.wireless status
```

## Related

- [Cudy AP3000 v1: OpenWrt flashing guide](cudy-ap3000-v1-openwrt.md)
- [Cudy AP3000 Wall v1: OpenWrt flashing guide](cudy-ap3000-wall-v1-openwrt.md)

## Sources

- [Flashing Cudy AP3000 Outdoor V1 — OpenWrt Forum](https://forum.openwrt.org/t/flashing-cudy-ap3000-outdoor-v1/235051)
- [Support for the Cudy AP3000 Outdoor — OpenWrt Forum](https://forum.openwrt.org/t/support-for-the-cudy-ap3000-outdoor/206332)
- [Cudy OpenWrt Software Download (FAQ)](https://www.cudy.com/en-us/blogs/faq/openwrt-software-download)
- [Legacy wiki.js: OpenWrt AP hardware device model pages](../../../../../sources/network/access-points/2026-09-28-openwrt-device-model-docs.md)
