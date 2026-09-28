---
type: guide
title: Running AdGuard Home directly on OPNsense (mimugmail plugin)
description: 'Install and configure AdGuard Home directly on OPNsense via the mimugmail plugin repository: firewall rules, coexisting with the built-in resolver, and local *.local resolution.'
tags: [adguard-home, opnsense, dns, pi-hole, mimugmail]
status: draft
resource:
created: 2026-09-28T18:54:10Z
updated: 2026-09-28T18:54:10Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:54:10Z
verified: []
stale_after: 2027-09-28T18:54:10Z
sources:
- id: 2026-09-28-opnsense-adguard-home-mimugmail
  resource: 'private:/sources/network/dns/2026-09-28-opnsense-adguard-home-mimugmail.md'
relations: []
superseded_by:
---

# Running AdGuard Home directly on OPNsense (mimugmail plugin)

[AdGuard Home](https://adguard.com/en/adguard-home/overview.html) can run directly on an OPNsense
router instead of on a separate server, via the third-party `mimugmail` plugin repository. Running
it on the router means the blocking/filtering DNS resolver doesn't depend on any other host being
up, and needs no extra DNS-forwarding setup between a server and the firewall.

> [!info] AdGuard Home vs. Pi-hole
> Both are network-wide DNS-based ad/tracker blockers with built-in DHCP and per-client
> configuration. AdGuard Home additionally offers: a native Go binary (no web server/PHP
> dependency), encrypted upstream DNS (DNS-over-HTTPS/TLS/DNSCrypt) both as a client and a server,
> built-in phishing/malware and parental-control blocklists, forced safe-search, and the ability to
> run without root privileges. Pi-hole needs manual `lighttpd` configuration for HTTPS on its admin
> UI and lacks encrypted upstream DNS support out of the box.

## Prerequisites

- An OPNsense router, version 23.1.1 or newer.
- Access to the OPNsense web interface and its console (SSH or physical/serial).

## Steps

### 1. Add the plugin repository

The AdGuard Home plugin isn't in the official OPNsense plugin repository; it's distributed via the
third-party `mimugmail` repository. Add it from the OPNsense console:

```bash
fetch -o /usr/local/etc/pkg/repos/mimugmail.conf https://www.routerperformance.net/mimugmail.conf
```

### 2. Install the plugin

In the web UI: **System → Firmware → Plugins**, find and install the AdGuard Home plugin.

### 3. Basic configuration

**Services → AdGuardHome** lets you configure AdGuard Home. In one deployment, the `Primary DNS`
checkbox had to be enabled for it to take over resolution.

### 4. Open a firewall rule for the setup port

This step is missing from most third-party guides and is required: AdGuard Home's web setup
wizard needs an explicit firewall rule on the interface it's bound to, opening the port it listens
on (commonly `3000/tcp` for the initial setup UI). Without this rule the setup wizard is
unreachable even though the service is running.

### 5. Decide what happens to the existing resolver

If AdGuard Home is going to be the LAN's DNS server, the router's existing resolver (Unbound, by
default on OPNsense) needs to either be disabled or moved out of the way:

- **Disable it:** **Services → Unbound DNS → General**, untick `Enable`. Simplest option if
  nothing else depends on it.
- **Keep it as a forwarder:** leave `Enable` ticked but change its listen port (e.g. to `5353`).
  AdGuard Home doesn't know the router's DHCP-assigned addresses, so forwarding AdGuard's queries
  for local names to the still-running resolver on its new port lets local hostnames keep
  resolving. This needs its own firewall rule for the new port, same as the setup port above.

### 6. Open the setup wizard

```url
http://<router-ip>:3000
```

Complete AdGuard Home's own setup wizard. In one deployment, binding the actual DNS service to
port `53` did not work reliably; changing it to a different port (opened via its own firewall
rule) resolved it.

### 7. Resolve local (`.local`) hostnames through AdGuard Home

Local hostnames (e.g. `<host>.local`) won't resolve through AdGuard Home by default, since AdGuard
doesn't know the router's DHCP leases/reservations or ARP table. Add an upstream DNS entry in
AdGuard Home's DNS settings that forwards the `local` domain specifically to the router's own DNS
resolver (the one from step 5), using dnsmasq-style syntax, e.g.:

```
[/local/]<router-ip>:<forwarder-port>
```

This makes every `*.local` query go to the router's resolver, which already knows every DHCP
client (including static reservations) and every device discovered via ARP — so no extra
per-host entries are needed. Alternatively, AdGuard Home's own built-in DHCP server can be used
instead of the router's, in which case this forwarding rule is unnecessary because AdGuard then
tracks every device itself.

## Verify

- The AdGuard Home admin UI and setup wizard are reachable on their configured port from a LAN
  client.
- A LAN client using the router as its DNS server can resolve a known ad/tracker domain to
  `0.0.0.0` (blocked) and a local `*.local` hostname to its correct LAN IP.

## Troubleshooting

- **Setup wizard unreachable:** check the firewall rule for the setup port exists on the correct
  interface (the one AdGuard Home is bound to), not just the LAN interface in general.
- **DNS queries on port 53 don't work:** try a different listen port for AdGuard Home's DNS
  service and open a matching firewall rule — this was unreliable in the source deployment for
  unclear reasons.
- **`*.local` names don't resolve:** confirm the upstream forwarding entry (`[/local/]<ip>:<port>`)
  points at the *still-running* resolver's actual listen port, and that resolver has a route to
  the DHCP lease/reservation table (i.e. it's the router's own resolver, not an external one).

## Related

## Sources

- [Legacy wiki.js (de, translated): OPNsense - AdGuard Home](../../../../../sources/network/dns/2026-09-28-opnsense-adguard-home-mimugmail.md)
