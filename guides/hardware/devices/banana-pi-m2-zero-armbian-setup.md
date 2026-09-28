---
type: guide
title: 'Banana Pi M2 Zero: Armbian setup and USB OTG host mode fix'
description: Headless Armbian installation and the g_serial gadget-driver fix for reliable USB host mode on the Banana Pi M2 Zero (Allwinner H2+).
tags: [banana-pi, armbian, sunxi, allwinner, sbc, esp32, linux]
status: draft
resource:
created: 2026-09-28T17:01:29Z
updated: 2026-09-28T17:01:29Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:01:29Z
verified: []
stale_after: 2027-09-28T17:01:29Z
sources:
- id: 2026-09-28-banana-pi-m2-zero
  resource: 'private:/sources/hardware/devices/2026-09-28-banana-pi-m2-zero.md'
relations: []
superseded_by:
---

# Banana Pi M2 Zero: Armbian setup and USB OTG host mode fix

Headless-install a Banana Pi M2 Zero with Armbian and fix its single most common gotcha: USB host
mode silently stops working a few seconds after every boot unless one gadget driver is disabled.

## Hardware overview

The Banana Pi M2 Zero is a low-cost single-board computer in the Raspberry Pi Zero form factor,
useful for embedded/IoT projects, headless servers, or as a flashing/debug station for ESP32/ESP8266
boards.

| | |
|---|---|
| **SoC** | Allwinner H2+ (quad-core ARM Cortex-A7, Mali-400MP2 GPU) |
| **RAM** | 512 MB DDR3 |
| **Storage** | microSD card slot (no onboard eMMC) |
| **Networking** | 2.4 GHz WiFi b/g/n + Bluetooth 4.0 (AP6212/BCM43430 combo chip) — **no Ethernet port** |
| **USB** | One micro-USB **OTG** port (data + limited power to attached devices in host mode) and a separate micro-USB **DC_IN** port (power input only) — two distinct connectors, easy to mix up |
| **GPIO** | 40-pin header, physically compatible with the Raspberry Pi Zero form factor and HATs (verify pinout compatibility per project) |
| **Other** | CSI camera connector, IR receiver, status LED, reset/power buttons |
| **Display** | No HDMI on most variants — check your specific board revision |

## Prerequisites

- A microSD card and a way to write images to it (Armbian Imager, or `dd`/Etcher).
- Your WiFi SSID, passphrase, and 2-letter country code (a wrong country code can prevent the radio
  from associating at all).
- No Ethernet cable will help you here — this board has none. **Pre-configuring WiFi and SSH before
  first boot is not optional**: the USB OTG port cannot reliably be used for a keyboard until after
  first boot (and even then needs the fix in [USB OTG host mode](#usb-otg-host-mode) below), and
  most variants have no HDMI output either. Flashing a stock image without WiFi credentials and SSH
  enabled ahead of time leaves you with no way to reach the board at all.

## Steps

### 1. Flash and configure headlessly

**Option A — Armbian Imager (recommended).** [Armbian Imager](https://imager.armbian.com/) can bake
WiFi credentials, SSH keys, a user account, and timezone/locale directly into the image before
writing it — no monitor/keyboard needed for first boot.

1. Download Armbian Imager.
2. Select Vendor → Banana Pi → Board → BPI-M2 Zero → OS (a Debian/Armbian minimal image).
3. In the config/profile step, set: username + password or SSH public key (SSH key preferred);
   WiFi SSID + passphrase + country code; enable SSH; timezone and locale.
4. Flash to the microSD card.

**Option B — manual (dd/Etcher + `armbian_first_run.txt`).** If Armbian Imager doesn't list this
board:

```bash
# Download an Armbian image for this board from armbian.com, then flash it
xz -dc Armbian_*_Bananapim2zero_*.img.xz | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync
# replace /dev/sdX with the actual SD card device — verify with lsblk first
```

Before ejecting, mount the boot partition and configure headless WiFi:

```bash
sudo mount /dev/sdX1 /mnt
cd /mnt
cp armbian_first_run.txt.template armbian_first_run.txt
```

Edit `armbian_first_run.txt`:

```ini
FR_general_delete_this_file_after_completion=1
FR_net_change_defaults=1
FR_net_ethernet_enabled=0
FR_net_wifi_enabled=1
FR_net_wifi_ssid='<your-ssid>'
FR_net_wifi_key='<your-wifi-password>'
FR_net_wifi_countrycode='<your-country-code>'
```

With this manual path, the default login is `root` / `1234`, which forces an immediate password
change and walks through creating a normal user.

### 2. Boot and connect

1. Unmount, insert the card, and power the board via the **DC_IN** micro-USB port — not the OTG port
   (OTG is data-focused; DC_IN is the dedicated power input).
2. Wait roughly 1–2 minutes for first boot (WiFi association plus first-run scripts take longer than
   a plain boot).
3. Find the IP via your router's DHCP client list, or try `ping bananapim2zero.local` <!-- wiki:allow --> (Armbian ships
   Avahi/mDNS by default — this is Armbian's generic default mDNS name for this board model, not a private hostname).
4. SSH in with the credentials set above.

### 3. Update after first boot

```bash
sudo armbian-config   # System > check for kernel/board updates
sudo apt update && sudo apt full-upgrade
```

## USB OTG host mode

The Banana Pi M2 Zero has a single physical USB0 port that is multiplexed between peripheral
(gadget) mode and host mode. Armbian's default configuration loads the `g_serial` gadget driver,
which claims the PHY for gadget mode a few seconds after boot, locking the port out of host mode
until reboot.

### Fix

Comment out `g_serial` in `/etc/modules` and reboot:

```bash
sudo sed -i 's/^g_serial/#g_serial/' /etc/modules
sudo reboot
```

Host mode then works reliably on subsequent plug/unplug cycles.

**Trade-off:** disabling `g_serial` also disables the USB gadget serial console (`ttyGS0`). If you
relied on a serial-over-USB connection to the board from a PC for debugging/console access, you lose
that capability. For most use cases this trade-off is acceptable.

### Symptoms

- USB devices enumerate briefly right after system boot.
- After unplugging a device, plugging it back in produces no enumeration event.
- The device never appears in `lsusb` on subsequent connections, even after rebooting.
- Kernel messages show no errors initially — the device is simply not detected.

### Root cause

Armbian's default `/etc/modules` loads the `g_serial` kernel module, which claims the SoC's shared
USB PHY for **gadget mode** (device/peripheral mode) several seconds after boot — visible in `dmesg`
as `Reached target usb-gadget.target`.

The Allwinner H2+ has a **single physical USB0 PHY that is multiplexed** between two paths: a
peripheral-only path (MUSB gadget controller) and the EHCI/OHCI host controllers. Once `g_serial`
claims the PHY for gadget mode, the port stays locked into that mode until the next reboot — which
is why host mode appears to work briefly during the boot window (the ~8–10 seconds before the gadget
target is reached) but fails afterward.

This architecture (single muxed OTG PHY) is shared by other Allwinner SoC boards — Orange Pi Zero,
Banana Pi Zero, and similar — so the same diagnostics and fix generally apply there too.

## Verify

```bash
# Watch boot messages for PHY muxing and the gadget target
dmesg | grep -i usb
```

Expected output includes:
- `Disabling musb host side code due to re-routed phy` — normal and expected; PHY is muxed to
  EHCI/OHCI for host mode.
- `Reached target usb-gadget.target` — the moment the port is claimed for gadget mode (should no
  longer matter once `g_serial` is disabled).

```bash
lsusb -t
lsusb
systemctl is-active usb-gadget.target
lsmod | grep g_
```

Confirmed working peripherals after the fix: a wireless keyboard/mouse USB combo receiver (2.4 GHz
dongle, full HID functionality) and an ESP32 development board (enumerates as `/dev/ttyACM0`,
"Espressif USB JTAG/serial debug unit" — usable for flashing ESP32/ESP8266 devices directly from the
board).

## Troubleshooting

If a USB device (e.g. a USB stick) still fails to enumerate even after disabling `g_serial`, and you
see dmesg errors like `port 1 reset error -110` or `Cannot enable, maybe cable is bad`, this
typically indicates a **power headroom issue** — the single micro-USB connector must supply both
board power and device power. Try:

- A self-powered USB hub connected to the OTG port.
- A lower-power device (e.g. a wireless keyboard receiver or an unpowered flash drive).
- A power supply that provides adequate current.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: Banana Pi M2 Zero](../../../../../sources/hardware/devices/2026-09-28-banana-pi-m2-zero.md) — private source (original capture)
- [Armbian Banana Pi M2 Zero page](https://www.armbian.com/banana-pi-m2-zero/)
- [Allwinner H2+ / sun8i reference (linux-sunxi wiki)](https://linux-sunxi.org/H2)
- [Linux MUSB controller documentation](https://www.kernel.org/doc/html/latest/usb/musb.html)
