---
type: guide
title: 'Cudy AP3000 Wall v1: OpenWrt flashing guide'
description: Hardware specs and how to flash OpenWrt onto a Cudy AP3000 Wall v1 access point, including a radio-band-numbering gotcha and an unusually long first-boot reboot.
tags: [openwrt, router, cudy, ap3000, hardware, flashing]
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

# Cudy AP3000 Wall v1: OpenWrt flashing guide

Wall-mount indoor (but not outdoor-rated) WiFi 6 access point on MediaTek's MT7981 filogic
platform. Sibling device to the [Cudy AP3000 v1](cudy-ap3000-v1-openwrt.md) (tabletop indoor) and
[Cudy AP3000 Outdoor v1](cudy-ap3000-outdoor-v1-openwrt.md) — **do not mix up their firmware
images**, they use different board IDs and sysupgrade will (correctly) refuse the wrong one.

## Hardware specifications

| Property | Value |
|---|---|
| SoC | MediaTek MT7981 (filogic) |
| CPU | 2x ARM Cortex-A53 (aarch64) |
| RAM | 512MB |
| Flash | 256MB NAND |
| Ethernet | 1x 2.5Gbit LAN |
| Power | 12V DC |
| WiFi | 2.4GHz + 5GHz, 802.11ax |
| Board ID | `cudy,ap3000wall-v1` |
| Vendor SKU | R68 |
| OpenWrt target | `mediatek/filogic` |

## Default network (stock Cudy firmware)

- Web UI / router IP: `192.168.10.254`  <!-- wiki:allow -->

## Steps: flashing OpenWrt

Cudy ships these with an OpenWrt-derived vendor firmware rather than a from-scratch custom OS, so
there's no exploit needed — but a direct jump from stock straight to the official OpenWrt
sysupgrade image is usually rejected. Go through Cudy's own "transition" image first. Some units
arrive pre-installed with the transition firmware already (observed: `23.05-SNAPSHOT-CUDY`).

### 1. Get the Cudy-signed transition image (if not already installed)

Cudy publishes an OpenWrt-based transition/intermediate firmware (not on the standard OpenWrt
downloads site — check Cudy's support Google Drive /
[cudytech.com](https://www.cudy.com/en-us/blogs/faq/openwrt-software-download) for the current
link). This image is signed so the stock firmware's web UI will accept it.

### 2. Flash the transition image via the stock web UI (if needed)

If not already on the transition firmware:

1. Log into the stock web UI at `192.168.10.254`.  <!-- wiki:allow -->
2. Upload the Cudy transition firmware through the normal firmware-update page.
3. Wait for reboot — the device now runs an OpenWrt-based intermediate firmware.

### 3. Flash the official OpenWrt sysupgrade

Once on the transition firmware, download the official OpenWrt **sysupgrade** image for board
`cudy_ap3000wall-v1` from the [OpenWrt downloads page](https://downloads.openwrt.org/releases/) and
flash it via `sysupgrade` (SSH/CLI) or LuCI — same as any normal OpenWrt version upgrade from here
on.

## Important: radio band numbering gotcha

**On this hardware, `radio0` = 2.4GHz and `radio1` = 5GHz** — the opposite of some other AP models
(e.g. some devolo hardware has 5GHz on `radio0`, 2.4GHz on `radio1`). When copying UCI
configuration from another device or troubleshooting wireless settings, **always verify the band
assignment via the `wireless.radioN.band=` value, never assume the radio numbering carries over**
from a sibling device or the same hardware in a different location.

Check with:

```sh
uci show wireless | grep "\.band="
```

Expected output:

```
wireless.radio0.band=2g
wireless.radio1.band=5g
```

If you see them reversed, there's been a hardware/driver quirk — don't blindly copy config, update
the band assignments to match.

## First-boot gotcha: longer-than-usual reboot

This hardware (especially when flashing official OpenWrt for the first time onto new NAND/UBI) can
take notably longer to return after a sysupgrade reboot compared to other MediaTek filogic APs —
up to ~18 minutes observed, vs. 2-3 minutes typical. **LEDs stay steady (normal boot behavior)** the
entire time; this is a **normal first-boot NAND initialization sequence**, not a boot loop or hang.
Do not power-cycle or assume failure until you've waited ~20 minutes and verified sshd is not
responding (or has come back online). After the initial boot, subsequent reboots are normal speed.

## Fallback: TFTP recovery

Requires stock firmware ≥ 2.4.7:

1. Set your PC to static IP `192.168.1.88`.  <!-- wiki:allow -->
2. Run a TFTP server serving the firmware file named `recovery.bin`.
3. Hold the device's reset button while powering on, release once the upload starts.

## Verify

```sh
cat /etc/openwrt_release
uci show wireless | grep "\.band="
```

## Related

- [Cudy AP3000 v1: OpenWrt flashing guide](cudy-ap3000-v1-openwrt.md)
- [Cudy AP3000 Outdoor v1: OpenWrt flashing guide](cudy-ap3000-outdoor-v1-openwrt.md)

## Sources

- [OpenWrt Table of Hardware: Cudy AP3000 Wall v1](https://openwrt.org/toh/cudy/ap3000_wall_v1)
- [Cudy OpenWrt Software Download (FAQ)](https://www.cudy.com/en-us/blogs/faq/openwrt-software-download)
- [Legacy wiki.js: OpenWrt AP hardware device model pages](../../../../../sources/network/access-points/2026-09-28-openwrt-device-model-docs.md)
