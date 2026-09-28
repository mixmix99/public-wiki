---
type: concept
title: Wake-on-LAN from a container on a Docker bridge network
description: Why a WoL magic packet sent from inside a Docker bridge-networked container never reaches the LAN, and the workarounds (host networking, macvlan, or a host-networked relay service).
tags:
- docker
- wake-on-lan
- wol
- networking
- home-assistant
- containers
status: draft
resource:
created: 2026-09-27T19:54:27Z
updated: 2026-09-28T17:06:25Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:06:25Z
verified: []
stale_after: 2028-09-26T19:54:27Z
sources:
- id: docker-bridge-wake-on-lan
  resource: private:/sources/home-automation/homeassistant/2026-09-27-docker-bridge-wake-on-lan.md
- id: upsnap-github
  resource: https://github.com/seriousm4x/UpSnap
relations: []
superseded_by:
---

# Wake-on-LAN from a container on a Docker bridge network

A Wake-on-LAN (WoL) magic packet is a **UDP broadcast** to the target subnet's broadcast address (typically `<subnet>.255:9`). An application (e.g. a home automation platform) running inside a container on Docker's default **bridge** network sits behind Docker's NAT, and that broadcast packet never reaches the physical LAN — so any WoL integration that works fine when run directly on the host silently does nothing from inside such a container.

## How it works

Docker's bridge networking gives a container its own private subnet (Docker's own default bridge range is `172.17.0.0/16` <!-- wiki:allow --> — a vendor default, not a real deployment's addressing) and NATs outbound traffic through the host. A directed broadcast (`x.x.x.255`) sent from inside the container is scoped to *that* virtual subnet, not the host's LAN — and even if it were somehow scoped correctly, Linux does not forward directed broadcasts across a NAT boundary by design. The magic packet simply never leaves the container's virtual network; there is usually no error, since UDP is fire-and-forget — the application believes it sent the packet successfully.

This differs from most other outbound traffic a containerized app makes (HTTP calls, DNS, etc.), which is ordinary unicast and NATs through cleanly. WoL is one of the few common cases where a container's network mode actually changes correctness, not just performance or IP visibility.

## When to use it / trade-offs

Three practical ways to fix it, in order of how much they disturb an existing container setup:

- **Run the specific service with `network_mode: host`.** The container gets the host's real network namespace, so a broadcast it sends is a real LAN broadcast. Simplest fix if the whole application can tolerate host networking (loses Docker's network isolation and port-mapping for that container).
- **Attach the container to a macvlan (or similar Layer-2) network** instead of the default bridge, giving it its own MAC/IP directly on the LAN. Broadcasts work correctly. More setup and host-networking-stack support required than `network_mode: host`, and doesn't always coexist cleanly with the host's own network stack (macvlan interfaces typically can't talk to the host itself without extra configuration).
- **Delegate WoL to a small, separate host-networked relay service**, and have the bridge-networked application call that relay over ordinary unicast HTTP instead of sending the magic packet itself. This avoids changing the main application's network mode at all — only the small relay needs host networking. A relay service can also usefully centralize device state checks (e.g. polling a device's own status API) and shutdown commands alongside the wake function, rather than each Home Assistant automation reimplementing WoL logic per device.

The relay approach is a good default when the main application is otherwise working well on bridge networking and you don't want to weaken its network isolation just for one feature.

## Pitfalls

- **Don't assume ICMP ping is a valid "is it awake" state check** for a target device that keeps its network interface up during standby — some smart TVs and similar devices respond to ping even while effectively "off". Use an application-level state endpoint (if the device exposes one) instead, and match a specific value, not just reachability.
- **A relay's templated command fields may not all get variable substitution.** If using a templating relay service, verify which fields actually receive placeholder substitution (e.g. device IP/MAC) versus which expect literal values — using a placeholder in a field that doesn't support it can fail silently, since the resulting command (e.g. an unresolvable hostname) just times out with no distinguishing log entry.
- **Don't assume a second "sleep" command wakes a device already in standby.** Some devices treat a repeated "go to standby" command as pushing them into a *deeper* sleep state rather than toggling power — verify wake and sleep are asymmetric operations for the specific device before building conditional logic around them.

## Related

- [Controlling a Philips Saphi TV via JointSpace and Home Assistant, with WoL fallback](../../../guides/home-automation/devices/philips-saphi-tv-jointspace-api.md) — a concrete device integration that hits this exact problem and applies the relay-service workaround.

## Sources

- [Private source: TV Wake-on-LAN from a Docker bridge network, fixed via UpSnap](../../../../../sources/home-automation/homeassistant/2026-09-27-docker-bridge-wake-on-lan.md) — private source (real incident)
- [UpSnap](https://github.com/seriousm4x/UpSnap) — example of a host-networked relay service implementing this pattern
