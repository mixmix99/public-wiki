---
type: guide
title: 'PiKVM: CLI maintenance reference'
description: 'Common PiKVM CLI maintenance tasks: read-only filesystem toggling, password/hostname changes, OS updates, static IP, ISO transfer, NFS image storage, and OLED setup.'
tags: [pikvm, kvm, arch-linux-arm, raspberry-pi]
status: draft
resource:
created: 2026-09-28T17:01:31Z
updated: 2026-09-28T17:01:31Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:01:31Z
verified: []
stale_after: 2027-09-28T17:01:31Z
sources:
- id: 2026-09-28-pikvm
  resource: 'private:/sources/hardware/kvm/2026-09-28-pikvm.md'
relations: []
superseded_by:
---

# PiKVM: CLI maintenance reference

A grab-bag reference for the CLI maintenance tasks that come up repeatedly with a
[PiKVM](https://pikvm.org/) IP-KVM appliance: changing default credentials, updating, networking,
image transfer, and adding an OLED status display.

## Prerequisites

- SSH access to the PiKVM (default `root`/`root` if unchanged — a documented vendor default, change
  it as the first step below).
- PiKVM's root filesystem is **read-only by default** (it runs Arch Linux ARM). Almost every task
  below follows the same pattern: make it writable, edit, make it read-only again:

  ```bash
  rw
  # ... edit / run commands ...
  ro
  ```

  Leaving the filesystem in `rw` longer than necessary risks SD-card corruption on power loss —
  always `ro` again once done.

## Steps

### Change the SSH password

Default SSH login is `root`/`root`. Change it with the standard `passwd` command:

```bash
rw
passwd root
ro
```

### Change the web UI password

Default web UI login is `admin`/`admin`. Change it with PiKVM's own helper (not the same account or
mechanism as the SSH password — the two are independent):

```bash
rw
kvmd-htpasswd set admin
ro
```

### Change the hostname

```bash
rw
hostnamectl set-hostname <hostname>
nano /etc/kvmd/meta.yaml
ro
```

The web UI's displayed name is read from `/etc/kvmd/meta.yaml`, so `hostnamectl` alone is not
enough — edit that file too if you want the web UI to reflect the new name.

### Update PiKVM

```bash
pikvm-update
```

If you hit `pikvm-update: command not found`, you're most likely on an old OS release. Bootstrap the
updater first:

```bash
rw
pacman -Syu
pacman -S pikvm-os-updater
pikvm-update
ro
```

After this one-time bootstrap, plain `pikvm-update` works going forward.

### Set up a static IP

Edit the relevant `systemd-networkd` unit — PiKVM uses `systemd-networkd`, not NetworkManager:

```bash
rw
nano /etc/systemd/network/eth0.network   # or wlan0.network for WiFi
ro
```

```ini
[Network]
Address=192.0.2.10/24
Gateway=192.0.2.1
DNS=192.0.2.1
DNS=198.51.100.1
```

If you're on WiFi and don't yet have a `/etc/systemd/network/wlan0.network` file, you'll first need
to migrate the WiFi settings from `netctl` to `systemd-networkd`.

### Copy ISO images via SSH

Much faster than uploading through the web UI. Images live in `/var/lib/kvmd/msd`; the mass-storage
device filesystem needs its own remount step (separate from the root-fs `rw`/`ro` toggle above):

```bash
kvmd-helper-otgmsd-remount rw
scp /path/to/iso/file.iso root@<pikvm-ip>:/var/lib/kvmd/msd/
# or:
rsync -av --progress /path/to/iso/file.iso root@<pikvm-ip>:/var/lib/kvmd/msd/
```

### Mount NFS storage for images

Share ISO images across an entire fleet of PiKVMs via NFS; images can then be uploaded to the NFS
share via the web UI while local storage remains available too. The `kvmd` user needs at least read
access to the NFS directories; write access is optional. Use the `soft` and `nolock` mount options
for best performance.

```bash
rw
pacman -S nfs-utils
kvmd-helper-otgmsd-remount rw
mkdir -p /var/lib/kvmd/msd/NFS_Primary
mkdir -p /var/lib/kvmd/msd/NFS_Secondary
kvmd-helper-otgmsd-remount ro
nano /etc/fstab
```

Example `/etc/fstab` entries:

```
<nfs-server>:/srv/nfs/NFS_Primary     /var/lib/kvmd/msd/NFS_Primary     nfs  vers=3,timeo=1,retrans=1,soft,nolock  0 0
<nfs-server>:/srv/nfs/NFS_Secondary   /var/lib/kvmd/msd/NFS_Secondary   nfs  vers=3,timeo=1,retrans=1,soft,nolock  0 0
```

Apply with a reboot:

```bash
reboot
```

If images are added to the NFS storage from outside PiKVM, refresh the web UI's list via
`Drive → Reset`.

### Add an OLED status display

PiKVM can drive an i2c OLED display (e.g. a 0.91" module) for basic status output.

1. Enable i2c:
   - Add `dtparam=i2c_arm=on` to `/boot/config.txt`.
   - Add `i2c-dev` to `/etc/modules-load.d/kvmd.conf`.
2. Enable the OLED systemd units:

```sh
systemctl enable --now kvmd-oled kvmd-oled-reboot kvmd-oled-shutdown
```

## Verify

- SSH/web UI logins succeed with the new credentials, and the old defaults no longer work.
- `hostnamectl status` and the web UI both show the new hostname.
- `pikvm-update` runs without `command not found`.
- `ip a` on the configured interface shows the static address; the PiKVM is reachable at it.
- Uploaded ISOs appear in the web UI's mass-storage device list.
- For NFS: `mount | grep kvmd/msd` shows the shares mounted after reboot.
- The OLED shows PiKVM's status screen after enabling the units.

## Troubleshooting

- **`pikvm-update: command not found`** — you're on an old OS release; bootstrap via
  `pacman -S pikvm-os-updater` first (see above).
- **Static IP not applying** — confirm you edited the correct interface's `.network` file
  (`eth0.network` vs. `wlan0.network`) and that WiFi setups have actually migrated from `netctl` to
  `systemd-networkd` first.
- **Web UI still shows the old device name after `hostnamectl`** — the web UI reads
  `/etc/kvmd/meta.yaml` separately; edit it too.
- **Forgot to `ro` after an edit** — re-run `ro` as soon as you notice; an SD card left in `rw` for
  long periods risks corruption on unexpected power loss.

## Related

- [NanoKVM REST API: authentication and power control](nanokvm-rest-api-power-control.md) — the
  other common IP-KVM platform; PiKVM's ATX API (`POST /api/atx/power?action=on`) is state-aware and
  uses plain HTTP Basic Auth, unlike NanoKVM's JWT-cookie flow.

## Sources

- [Legacy wiki.js: PiKVM CLI and maintenance reference](../../../../../sources/hardware/kvm/2026-09-28-pikvm.md) — private source (original capture)
