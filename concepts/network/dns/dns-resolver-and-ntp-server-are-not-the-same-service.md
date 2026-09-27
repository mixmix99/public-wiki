---
type: concept
title: A DNS resolver's IP is not automatically an NTP server
description: Why advertising a host as both DNS and NTP server via DHCP silently fails clients if nothing on that host actually listens on UDP/123.
tags: [dns, ntp, dhcp, chrony, systemd-timesyncd, monitoring]
status: draft
resource:
created: 2026-09-27T19:49:16Z
updated: 2026-09-27T19:49:16Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T19:49:16Z
verified: []
stale_after: 2028-09-26T19:49:16Z
sources:
- id: 2026-09-27-local-dns-adguard
  resource: 'private:/sources/network/dns/2026-09-27-local-dns-adguard.md'
relations: []
superseded_by:
---

# A DNS resolver's IP is not automatically an NTP server

A DHCP server can hand out the same host's IP for two different purposes — DNS server (option 6)
and NTP server (option 42) — without anything actually verifying that the host runs both
services. If it only runs one, clients using the other fail silently, with no error visible unless
you go looking for it.

## How it works

DHCP's DNS and NTP options are independent, unrelated address hints — the DHCP server just repeats
whatever addresses it's configured with. There is no protocol-level check that a service is
actually listening at the address advertised for it. It's easy to set up one server (e.g. an
ad-blocking DNS resolver like AdGuard Home or Pi-hole) on a host, then later reuse "that host's
IP" for NTP too — in a router's DHCP config, or by copy-pasting a working DNS entry — without
separately installing and enabling an NTP daemon on it.

The failure is silent by design: `systemd-timesyncd` (the common lightweight NTP client on modern
Linux) treats a non-responding NTP server as a routine timeout, not an error, and just backs off
its poll interval — up to roughly 30+ minutes between attempts. Nothing crashes, nothing logs
loudly by default, and the host keeps running on its existing (increasingly wrong) clock.

```
systemd-timesyncd[...]: Timed out waiting for reply from <ip>:123 (<hostname>).
```

`timedatectl status` will show `System clock synchronized: no` if you check it directly, but
nothing prompts you to check.

## When to use it / trade-offs

- **Advertising the same host for both DNS and NTP is fine and common** — many small NTP daemons
  (chrony, ntpd) are lightweight enough to run alongside a DNS resolver on the same box. The
  pitfall isn't the pattern, it's assuming a service exists because its address was advertised.
- **chrony** is a straightforward way to turn a Linux host into an NTP server for a LAN: install
  it, add an `allow <subnet>` rule, restart. It also happily runs as the host's own NTP client at
  the same time.
- If you don't want to run your own NTP server, don't advertise your DNS host's IP for NTP at all
  — point DHCP's NTP option at public pool servers directly instead.

## Pitfalls

- **The failure mode looks unrelated to its cause.** A host with drifting time doesn't announce
  "my NTP source doesn't work" — the downstream symptom shows up wherever something is sensitive
  to absolute timestamps: a monitoring dashboard marking an otherwise-healthy host "stale" or
  "offline" because its last-seen timestamp is minutes in the past, TLS/auth failures with
  clock-skew-sensitive protocols, or scheduled jobs firing at the wrong wall-clock time.
- **Check `timedatectl status` and the client's NTP logs directly** rather than trusting that
  "DHCP says there's an NTP server, so time sync must be fine." `System clock synchronized: no`
  is the tell.
- **A client-side manual NTP override (pointing at public pools) can mask the underlying
  misconfiguration** rather than fixing it — useful as an immediate per-host fix, but the DHCP
  advertisement is still wrong for every other client on the network unless the actual server is
  fixed (or the advertisement removed).
- **Resetting an existing NTP server list in `systemd-timesyncd` config needs an explicit empty
  entry first** (`NTP=` on its own line before the replacement list) — otherwise a
  DHCP-provided entry from a generated drop-in just gets appended to, and is tried before your
  override.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: Local DNS (nexus / AdGuard Home)](../../../../../sources/network/dns/2026-09-27-local-dns-adguard.md) — private source (real incident)
