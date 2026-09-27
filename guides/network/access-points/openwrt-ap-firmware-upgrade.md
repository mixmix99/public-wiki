---
type: guide
title: Upgrading OpenWrt access point firmware safely
description: 'Generic checklist for upgrading OpenWrt AP firmware: backing up config first, upgrading one device at a time, and recovering from a config wipe.'
tags: [openwrt, wifi, access-point, upgrade, sysupgrade]
status: draft
resource:
created: 2026-09-27T19:48:50Z
updated: 2026-09-27T19:48:50Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T19:48:50Z
verified: []
stale_after: 2027-09-27T19:48:50Z
sources:
- id: openwrt-ap-upgrade
  resource: private:/sources/network/access-points/2026-09-27-openwrt-ap-upgrade.md
relations: []
superseded_by:
---

# Upgrading OpenWrt access point firmware safely

`sysupgrade` (directly, or via a wrapper like `owut`) is documented to preserve `/etc/config/*`
across an OpenWrt firmware upgrade. In practice it doesn't always — a config can be silently wiped
even without the "wipe everything" `-n` flag. This checklist assumes that can happen and treats a
local backup as mandatory, not optional.

## Prerequisites

- SSH key access to the device(s) being upgraded.
- A local folder to keep backup archives until each upgrade is confirmed good.
- If upgrading several devices: do them one at a time, not all in parallel (see [Steps](#steps)).

## Steps

### 1. Back up the config first — download it to your machine

Don't rely on `sysupgrade`'s built-in config preservation alone.

```sh
# Per device — creates a real sysupgrade backup archive and copies it off
ssh root@<device-ip> "sysupgrade -b /tmp/backup.tar.gz"
scp root@<device-ip>:/tmp/backup.tar.gz "./backups/<device-name>-$(date +%F).tar.gz"
```

Keep these until the upgrade is confirmed good on every device.

### 2. Check current & target version before flashing

```sh
ssh root@<device-ip> "cat /etc/openwrt_release; owut check"
```

`owut check` (or your upgrade tool's equivalent) shows the target version and whether any installed
packages have known build failures against it — read the output rather than skipping past it.

### 3. Upgrade one device at a time, not the whole fleet blind

Firing an upgrade command across many devices in parallel makes failures hard to diagnose: a bug in
the upgrade tool can fail silently on some devices, and there is no way to tell which ones actually
flashed without checking each individually afterward. Upgrade in small batches or one at a time,
and read the actual command output for each device.

**Known failure modes to watch for (generic to `owut`/`sysupgrade`-style tooling):**

| Symptom | Likely cause | What to do |
|---|---|---|
| A null-reference / internal error during download | Bug in the upgrade tool itself | Nothing was flashed yet — device is untouched. Retry, or fall back to downloading the firmware image manually and running `sysupgrade -v <file>.bin` directly. |
| A `ubus`/RPC error during a post-flash verify step | Same class of tooling bug | Same as above — check whether anything actually flashed before assuming the worst. |
| Connection to the firmware server fails | Device clock isn't NTP-synced (common right after first boot, or after a long power-off) | Check `date` on the device — a wildly wrong clock breaks HTTPS certificate validation. Fix network/NTP first, then retry. |
| SSH session drops mid-upgrade | Normal — the upgrade process reboots the device | Expected. Wait for it to come back (step 4). |

### 4. Wait for reboot — and expect the IP to change if the device is a DHCP client

- Faster/newer hardware typically comes back within a few minutes; older/slower hardware can take
  5-10 minutes.
- Prefer mDNS (`<hostname>.local`) over a remembered IP — it survives an IP change.
- If truly unreachable after ~10 minutes, check whether the device fell back to its firmware's
  factory-default address (OpenWrt's is `192.168.1.1`, a vendor default). <!-- wiki:allow --> That means the config was wiped — see step 5.
  Reaching a device at a factory-default address safely requires adding a **secondary** IP to your
  own machine in that subnet, not replacing your primary address (which would drop your own
  internet connectivity). On Windows, `netsh interface ipv4 add address` adds a secondary address
  without touching the DHCP-assigned primary one; a cmdlet that reconfigures the whole interface
  (e.g. one that sets a static address) can silently disable DHCP on it instead.

### 5. If the config got wiped — recovery checklist

If a device comes back with its firmware's default SSID/hostname and/or default IP, the upgrade ate
its config. Recover in order:

1. **Get it back on the network**: reset the LAN interface to a DHCP client and remove any static
   IP left over from the factory default.
2. **Restore network/wireless/system config** — from your step-1 backup if you have one, or by
   cloning the config from a sibling device of the *same hardware model* and adapting the
   hostname/SSIDs.
3. **Re-apply any security-relevant defaults you rely on** (e.g. IPv6 disabled, VLAN layout) — a
   cloned sibling config may itself predate a policy you've since adopted fleet-wide. Don't assume
   a working sibling config is fully up to date.
4. **Reinstall any packages the base firmware doesn't include** (roaming daemons, mDNS, etc.) —
   this typically needs the device's clock to be correct first, since package managers validate
   TLS certificates against the current time.
5. **Re-add your SSH key explicitly.** After a factory reset, some SSH daemons (e.g. dropbear with
   no root password set) accept a passwordless `none`-auth method that can make agent-based clients
   connect successfully *without the key actually being installed* — this can mask a missing key
   until a stricter client (one that insists on explicit publickey auth) fails later. Don't treat
   "I could SSH in" as proof the key is installed; write it to the authorized-keys file explicitly
   as a standard step.
6. **Redeploy any TLS certificates** — a wiped config typically wipes the web server's certificate
   files and UCI settings too, even though a base firmware upgrade doesn't otherwise touch them.
   If certificates are managed/cached centrally, redeploying usually doesn't need a new certificate
   issuance, just re-pushing the existing one.
7. Reload the wireless subsystem and verify SSIDs are broadcasting again.

## Verify

```sh
ssh root@<device-ip> "cat /etc/openwrt_release | grep DISTRIB_RELEASE; uci get system.@system[0].hostname; iwinfo 2>&1 | grep ESSID"
```

Checklist: correct firmware version · correct hostname · all expected SSIDs present · any policies
you rely on (e.g. IPv6 disabled) still hold · management interface (SSH/HTTPS) reachable without
warnings.

## Troubleshooting

See the failure-mode table in [step 3](#3-upgrade-one-device-at-a-time-not-the-whole-fleet-blind)
and the [recovery checklist](#5-if-the-config-got-wiped--recovery-checklist) above.

**Devices needing special handling:**
- A firmware version jump of more than one release can require a serial-console-assisted update on
  some hardware rather than a remote upgrade — check your device's upgrade notes before attempting
  a large jump remotely.
- Devices still on vendor stock firmware (not yet flashed to your target firmware at all) need a
  different, hardware-specific first-flash process, not this checklist.

## Related

- [OpenWrt multi-AP fleet policy](../../../concepts/network/access-points/openwrt-ap-fleet-policy.md) — the fleet-wide policy every device should return to after an upgrade

## Sources

- [Legacy wiki.js: OpenWrt AP upgrade process](../../../../../sources/network/access-points/2026-09-27-openwrt-ap-upgrade.md) — private source (real fleet this checklist was distilled from)
