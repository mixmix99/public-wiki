---
type: guide
title: Cross-flashing an LSI/Broadcom SAS 9300-16i HBA from IR to IT mode
description: Cross-flash a Broadcom/LSI SAS 9300-16i from IR (RAID) to IT (HBA passthrough) firmware using sas3flash, required for ZFS-based NAS software to get direct disk access.
tags: [hba, sas, zfs, firmware, broadcom, lsi]
status: draft
resource:
created: 2026-09-28T17:05:19Z
updated: 2026-09-28T17:05:19Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:05:19Z
verified: []
stale_after: 2027-09-28T17:05:19Z
sources:
- id: 2026-09-28-lsi-sas-9300-16i-it-mode-flash
  resource: 'private:/sources/hardware/components/2026-09-28-lsi-sas-9300-16i-it-mode-flash.md'
relations: []
superseded_by:
---

# Cross-flashing an LSI/Broadcom SAS 9300-16i HBA from IR to IT mode

ZFS (and NAS software built on it) wants direct, unmediated access to each disk. HBAs that ship in
**IR mode** (Integrated RAID) sit a RAID layer between the OS and the disks, which hides SMART
data, breaks TRIM, and can silently reorder/remap drives. This guide cross-flashes a
Broadcom/LSI **SAS 9300-16i** from IR to **IT mode** (Initiator-Target, i.e. plain HBA passthrough)
using the official `sas3flash` utility.

## Background: why the 9300-16i needs two flashes

The 9300-16i is not a single 16-port chip — it's two independent **SAS3008** controllers on one
PCB (each handling 8 ports), bridged behind a PCIe switch. `lspci` shows two separate SAS3008
devices:

```
04:00.0 Serial Attached SCSI controller: Broadcom / LSI SAS3008 ... (rev 02)
06:00.0 Serial Attached SCSI controller: Broadcom / LSI SAS3008 ... (rev 02)
```

Because firmware lives per-chip, **each controller must be flashed separately** — there is no
single "16i" firmware image; Broadcom's official 9300-8i IT firmware is exactly the right image
for each chip (a 9300-16i is, from the firmware's point of view, two 9300-8i controllers).

> Broadcom has removed SAS3008/9300-series firmware from its current support portal — the download
> search only lists the newer 24Gb Tri-Mode (9400+) family. The firmware itself is publicly
> redistributable under Broadcom's own license, and is mirrored by the community, e.g.
> [vanities/lsi-9300-8i-firmware](https://github.com/vanities/lsi-9300-8i-firmware) (includes both
> the firmware `.bin` and a Linux `sas3flash` binary).

## Prerequisites

- A Linux box with the card installed (a bench/test machine is fine — the card doesn't need to be
  in its final server yet).
- Root/sudo access.
- The `sas3flash` utility (Linux x64 static binary) and an IT-mode firmware image for the SAS3008
  chip.

## Steps

### 1. Identify the controllers

```bash
lspci -nnk | grep -A3 -i 'sas'
```

Confirm you see two SAS3008 entries with `Kernel driver in use: mpt3sas`. Check `dmesg` for the
currently loaded firmware:

```bash
dmesg | grep -i mpt3sas | grep -iE 'FWVersion|Protocol'
```

IR firmware and IT firmware both report `Protocol=(Initiator,Target)` in this line — that's just
the chip's SCSI target *capability*, not proof RAID is active. Don't use it to tell IR and IT
apart (see the verification step below).

### 2. Get the tool and firmware onto the box

```bash
curl -sL -o sas3flash "https://raw.githubusercontent.com/vanities/lsi-9300-8i-firmware/main/flasher/sas3flash"
curl -sL -o SAS9300_8i_IT.bin "https://raw.githubusercontent.com/vanities/lsi-9300-8i-firmware/main/firmware/SAS9300_8i_IT_16.00.12.00.bin"
chmod +x sas3flash
```

> Version `16.00.12.00` also fixes a known SATA reset/timeout bug present in older SAS3008
> firmware — worth having regardless of IR/IT mode.

### 3. List controllers

```bash
./sas3flash -listall
```

```
Num   Ctlr            FW Ver        NVDATA        x86-BIOS         PCI Addr
----------------------------------------------------------------------------
0  SAS3008(C0)  07.00.01.00    07.01.00.03    08.15.00.00     00:04:00:00
1  SAS3008(C0)  07.00.01.00    07.01.00.03    08.15.00.00     00:06:00:00
```

Two controllers, indexed `0` and `1` — both need flashing.

### 4. Flash each controller

```bash
./sas3flash -o -c 0 -f SAS9300_8i_IT.bin
./sas3flash -o -c 1 -f SAS9300_8i_IT.bin
```

- `-o` overrides the personality mismatch check (needed when crossing IR → IT).
- `-c <n>` targets a specific controller index from the `-listall` output.
- No BIOS/UEFI option ROM is flashed here on purpose — skip it if you don't need this card to be
  bootable. It's one less thing to flash, and the card skips its option-ROM scan at POST, speeding
  up boot slightly. If you do need to boot from drives behind this card, flash the matching
  BIOS/UEFI `.rom` files as a separate step.

Each successful run ends with:

```
Firmware Flash Successful.
Resetting Adapter...
Adapter Successfully Reset.
Finished Processing Commands Successfully.
```

## Verify

```bash
./sas3flash -listall
```

Both controllers should now show the new firmware version (e.g. `16.00.12.00`).

Reboot, then confirm no RAID/IR subsystem came up — this is the real IT-vs-IR signal, not the
`Protocol=` capability line:

```bash
dmesg | grep -iE 'non-optimal|IR volume|raid'
```

No output means no IR volume manager is running, i.e. clean IT mode. If drives are attached, they
should show up immediately as plain disks:

```bash
lsblk -o NAME,SIZE,MODEL,TRAN,ROTA,SERIAL
```

## Troubleshooting

- **Flash refuses to proceed / reports a personality mismatch:** make sure `-o` is passed — it's
  required specifically when crossing IR → IT, not just for a same-mode firmware update.
- **`dmesg` still shows `Protocol=(Initiator,Target)` after flashing and you're unsure if it
  worked:** that line reports chip capability on both IR and IT firmware — check for the *absence*
  of RAID/IR volume messages after reboot instead (see Verify above).
- **Can't find firmware on Broadcom's own site:** the SAS3008/9300 family has been removed from
  the current support portal in favor of newer Tri-Mode cards; use a community mirror of the
  original (publicly redistributable) firmware instead.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: LSI SAS 9300-16i IT-mode flash](../../../../../sources/hardware/components/2026-09-28-lsi-sas-9300-16i-it-mode-flash.md) — private source
