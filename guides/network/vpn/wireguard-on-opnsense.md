---
type: guide
title: Setting up WireGuard on OPNsense
description: Install the os-wireguard plugin, choose Road Warrior vs site-to-site, and work around WireGuard's lack of automatic reconnect and its split-tunnel DNS limitations on Windows clients.
tags: [wireguard, opnsense, vpn, dns]
status: draft
resource:
created: 2026-09-28T17:01:15Z
updated: 2026-09-28T17:01:15Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:01:15Z
verified: []
stale_after: 2027-09-28T17:01:15Z
sources:
- id: 2026-09-28-opnsense-wireguard-setup
  resource: 'private:/sources/network/vpn/2026-09-28-opnsense-wireguard-setup.md'
relations: []
superseded_by:
---

# Setting up WireGuard on OPNsense

Run a [WireGuard](https://www.wireguard.com/) VPN endpoint on an [OPNsense](https://opnsense.org/)
firewall via the official plugin, for either remote-client ("Road Warrior") or router-to-router
("Site-to-Site") tunnels — plus two gotchas that show up once the setup gets more advanced: no
automatic reconnect after a peer's IP changes, and DNS split-tunneling on Windows clients.

## Prerequisites

- An OPNsense firewall with access to System → Firmware → Plugins.
- For Road Warrior: at least one client device that will run the official WireGuard client.
- For Site-to-Site: a second WireGuard-capable router/host to peer with.

## Steps

### 1. Install the plugin

1. Open the OPNsense GUI.
2. Go to **System → Firmware → Plugins**.
3. Search for `os-wireguard` and click the **+** to install it.
4. Once installed, configuration lives under **VPN → WireGuard**.

### 2. Choose Road Warrior or Site-to-Site

- **Road Warrior** — a single client (laptop, phone) reaches the local network from anywhere.
  This is the default/standard WireGuard use case (one server, one or more client peers). OPNsense
  has a step-by-step wizard for it — see the [official
  how-to](https://docs.opnsense.org/manual/how-tos/wireguard-client.html).
- **Site-to-Site** — two full networks reach each other continuously, via two OPNsense (or other
  WireGuard-capable) routers peering with each other. More involved than Road Warrior since both
  sides need routing/interface configuration — see the [official
  how-to](https://docs.opnsense.org/manual/how-tos/wireguard-s2s.html).

### 3. Handle peers with a dynamic-IP endpoint (auto-reconnect)

WireGuard only resolves a peer's endpoint hostname once, at the initial handshake — if that peer's
IP later changes (e.g. it's behind a dynamic-DNS home connection), WireGuard keeps retrying the
stale old IP and never reconnects on its own. See [WireGuard endpoint DNS re-resolution](../../../concepts/network/vpn/wireguard-dns-reresolve.md) for the underlying cause and re-resolve mechanisms
(the OPNsense side of this is typically a periodic cron job that re-applies the peer's configured
hostname; Linux clients/servers have a ready-made `wireguard-tools` script and systemd timer for
the same purpose).

### 4. Split-tunnel DNS on Windows clients (optional, advanced)

By default, a WireGuard client's `DNS =` config line redirects **all** DNS queries through the
tunnel while it's up — there's no built-in way to route only specific domains' lookups through the
VPN and let everything else resolve normally, even by appending a DNS suffix to the `DNS =` line.

The documented workaround uses Windows' **NRPT** (Name Resolution Policy Table, a mechanism
originally built for DirectAccess) via `PostUp`/`PostDown` script hooks in the WireGuard for
Windows client's tunnel config:

```ini
[Interface]
PostUp = powershell -command "Add-DnsClientNrptRule -Namespace '<domain.tld>','.<domain.tld>' -NameServers '<internal-dns-server-ip>'"
PostDown = powershell -command "Get-DnsClientNrptRule | Where { $_.Namespace -match '.*\.domain\.tld' } | Remove-DnsClientNrptRule -force"
```

This adds an NRPT rule for `<domain.tld>` (and its subdomains) pointing at your internal DNS
server only while the tunnel is up, and removes it again on disconnect — every other DNS query
keeps using the client's normal resolver.

Script execution is disabled by default in the WireGuard for Windows client; enable it first via
registry:

```powershell
reg add HKLM\Software\WireGuard /v DangerousScriptExecution /t REG_DWORD /d 1 /f
```

## Verify

- **Road Warrior / Site-to-Site:** confirm the WireGuard handshake completes (OPNsense shows the
  peer's last handshake time under VPN → WireGuard → Status) and traffic routes as expected.
- **NRPT split-tunnel DNS:** while the tunnel is connected, run (elevated PowerShell)
  `Get-DnsClientNrptRule` and confirm the rule for `<domain.tld>` is present. Prefer
  `Resolve-DnsName <name>` or a browser/`ping` to test resolution — `nslookup` does not honor NRPT
  rules and will appear to fail even when the setup is correct.

## Troubleshooting

- **A peer stops reconnecting after its IP changes:** this is expected WireGuard behavior, not a
  misconfiguration — see [WireGuard endpoint DNS re-resolution](../../../concepts/network/vpn/wireguard-dns-reresolve.md) for the
  fix (a periodic re-resolve job).
- **`nslookup` fails to resolve a split-tunnel domain even though the tunnel and NRPT rule look
  correct:** this is a known NRPT limitation, not a broken setup — test with `ping`,
  `Resolve-DnsName`, or an application (browser) instead.
- **NRPT `PostUp`/`PostDown` scripts silently don't run at all:** check that
  `DangerousScriptExecution` is set in the registry (see step 4) — the WireGuard Windows client
  refuses to run `PostUp`/`PostDown` scripts otherwise.

## Related

- [WireGuard endpoint DNS re-resolution](../../../concepts/network/vpn/wireguard-dns-reresolve.md) — the general re-resolve
  problem and Linux-side fix, referenced from step 3 above.

## Sources

- [Legacy wiki.js: OPNsense WireGuard setup](../../../../../sources/network/vpn/2026-09-28-opnsense-wireguard-setup.md) — private source (original wiki.js text)
