---
type: concept
title: WireGuard endpoint DNS re-resolution
description: Why a WireGuard peer with a dynamic-IP endpoint hostname stops reconnecting after an IP change, and how to periodically re-resolve it on Linux and OPNsense.
tags:
- wireguard
- vpn
- dns
- dynamic-dns
- systemd
status: draft
resource:
created: 2026-09-28T17:01:18Z
updated: 2026-09-28T18:55:16Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:55:16Z
verified: []
stale_after: 2028-09-27T17:01:18Z
sources:
- id: 2026-09-28-wireguard-dns-reresolve
  resource: private:/sources/network/vpn/2026-09-28-wireguard-dns-reresolve.md
- id: 2026-09-28-wireguard-hetzner-cron-reresolve
  resource: private:/sources/network/vpn/2026-09-28-wireguard-hetzner-cron-reresolve.md
relations: []
superseded_by:
---

# WireGuard endpoint DNS re-resolution

[WireGuard](https://www.wireguard.com/) resolves a peer's `Endpoint =` hostname to an IP address
exactly once, at the initial handshake — it never re-resolves it afterward. If that peer sits
behind a dynamic (non-static) IP and the IP changes, WireGuard keeps sending packets to the old,
now-wrong address and the tunnel silently stops working until something forces a re-resolve.

## How it works

WireGuard's protocol design deliberately keeps the data plane minimal: once a peer's endpoint is
resolved to an IP, that IP is what's used for all further traffic, with no periodic DNS lookups
built in. This is fine for peers with a static IP, but breaks any peer configured with a dynamic-DNS
hostname (a home connection, a mobile client, cloud infrastructure with ephemeral IPs) — the tunnel
works until the far end's IP changes, then goes silently dead with no automatic recovery, even
though the DNS record itself is correctly updated.

## When to use it / trade-offs

- **Applies whenever at least one WireGuard peer's endpoint is a hostname (not a static IP)** that
  can change — most commonly a dynamic-DNS name for a home/road-warrior peer.
- **Not needed for peers with a genuinely static IP** — there's nothing to re-resolve.
- The fix is always an *external* periodic job that re-applies the peer's configured endpoint
  (forcing a fresh DNS lookup), not a WireGuard setting — this is a deliberate protocol design
  choice, not a bug, so there is no built-in flag to enable automatic re-resolution.

## Pitfalls

- **Assuming a tunnel drop is a routing/firewall problem** when the real cause is a stale resolved
  IP — check whether the peer's DNS record has changed since the last successful handshake before
  debugging anything else.
- **A re-resolve job must run frequently enough** relative to how often the peer's IP actually
  changes and how long the resulting outage is tolerable — the reference implementation below
  recommends roughly every 30 seconds.

## Implementations

### Linux: `wireguard-tools`' `reresolve-dns.sh` + systemd timer

The `wireguard-tools` package ships a contrib script,
`reresolve-dns.sh`, that re-resolves an interface's peer endpoint hostname(s) and updates the
running WireGuard config if the resolved IP changed. Run it via a systemd oneshot service plus a
timer, per WireGuard interface:

```ini
# /etc/systemd/system/wg-reresolve-dns@.service
[Unit]
Description=wg-reresolve-dns@

[Service]
Type=oneshot
ExecStart=/usr/share/doc/wireguard-tools/examples/reresolve-dns/reresolve-dns.sh %i
```

```ini
# /etc/systemd/system/wg-reresolve-dns@.timer
[Unit]
Description=wg-reresolve-dns@ timer

[Timer]
Unit=wg-reresolve-dns@%i.service
OnCalendar=*-*-*/*:*:00,30
Persistent=true

[Install]
WantedBy=timers.target
```

Enable per interface (repeat for each WireGuard interface that has a dynamic-endpoint peer):

```bash
systemctl enable --now wg-reresolve-dns@wg0.timer
```

(A third-party one-line installer script exists that sets up the same unit/timer pair
automatically — functionally equivalent to creating the files above by hand.)

**Cron alternative:** the systemd timer isn't required — a plain crontab entry works just as well
if the host doesn't run systemd or a coarser interval is acceptable:

```cron
*/15 * * * * /usr/share/doc/wireguard-tools/examples/reresolve-dns/reresolve-dns.sh wg0 >> /var/log/wg-reresolve.log 2>&1
```

Every 15 minutes is markedly coarser than the ~30 s systemd-timer interval above, so a peer's
outage after an IP change can last up to ~15 minutes with this variant instead of under a minute —
acceptable when the tunnel isn't latency- or availability-critical. The script itself only logs
when it actually changes an endpoint, so the log file stays quiet during normal operation.

#### How `reresolve-dns.sh` decides what to touch

The script is a line-by-line parser for the WireGuard `.conf` file, paired with a decision
function run per peer:

1. Parse each `[Peer]` block, collecting only its `PublicKey` and `Endpoint` (all other keys are
   ignored). The decision function runs at every new section header and again at end of file.
2. For each peer: skip it if it has no `PublicKey`+`Endpoint` pair, skip it if `wg` reports no
   handshake for it at all, and skip it if its last handshake is **not yet older than 135
   seconds**. Only a peer that fails all three skip checks gets re-resolved, via
   `wg set <interface> peer <key> endpoint <host:port>` — `<host:port>` still contains the
   original **hostname** from the config, so this call is what triggers the fresh DNS lookup, with
   no separate `dig`/`getent` step.
3. **The 135-second threshold** is deliberately set above WireGuard's default 25-second keepalive
   interval, so a peer that's merely quiet between keepalives is never mistaken for a dead one —
   only a peer with no successful handshake for well over one keepalive cycle gets touched.

**What the script deliberately does *not* do:** restart the WireGuard interface or drop any
existing connection, log anything when an endpoint update is *not* needed, or handle peers that
have no `Endpoint` configured at all (inbound-only/road-warrior clients, which have nothing to
re-resolve).

### OPNsense: periodic re-apply via cron

OPNsense doesn't ship an equivalent script out of the box; the documented approach is a
System → Settings → Cron job (e.g. every 15 minutes) that re-applies the WireGuard peer
configuration so its endpoint hostname gets re-resolved. See [Setting up WireGuard on
OPNsense](../../../guides/network/vpn/wireguard-on-opnsense.md) for the OPNsense-side setup this
applies to.

## Verify

After a re-resolve run, `wg show <interface>` (Linux) or the OPNsense WireGuard status page should
reflect a current "latest handshake" time close to now for the affected peer, even after its
underlying IP changed.

## Related

- [Setting up WireGuard on OPNsense](../../../guides/network/vpn/wireguard-on-opnsense.md) — the
  OPNsense-side guide that links here for the reconnect problem.

## Sources

- [Legacy wiki.js: WireGuard DNS re-resolve on Linux](../../../../../sources/network/vpn/2026-09-28-wireguard-dns-reresolve.md) — private source (original wiki.js text)
- [Legacy wiki.js (de, translated): WireGuard dynamic DNS reconnect (cron)](../../../../../sources/network/vpn/2026-09-28-wireguard-hetzner-cron-reresolve.md) — private source; cron alternative and script-internals reference
