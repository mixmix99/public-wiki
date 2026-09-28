---
type: concept
title: OpenWrt seamless Wi-Fi roaming with 802.11r/k/v and usteer
description: How 802.11k Neighbor Reports, 802.11r Fast BSS Transition, and 802.11v BSS Transition Management combine with a steering daemon like usteer to fix sticky Wi-Fi clients across a multi-AP network.
tags: [openwrt, wifi, roaming, ieee80211r, ieee80211k, ieee80211v, usteer]
status: draft
resource:
created: 2026-09-28T17:03:13Z
updated: 2026-09-28T17:03:13Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:03:13Z
verified: []
stale_after: 2028-09-27T17:03:13Z
sources:
- id: 2026-09-28-openwrt-seamless-roaming
  resource: 'private:/sources/network/wifi/2026-09-28-openwrt-seamless-roaming.md'
relations: []
superseded_by:
---

# OpenWrt seamless Wi-Fi roaming with 802.11r/k/v and usteer

In a multi-AP network where several access points broadcast the same SSID, clients decide on their
own when to roam. Phones and laptops are conservative by design: they only disconnect from the
current AP when the signal drops to nearly nothing, because reconnecting costs time and briefly
interrupts traffic. This causes the **sticky client problem** — a device standing next to a strong
AP stays connected to a distant, weak one until the connection is barely functional. Three Wi-Fi
amendments (802.11k/r/v) work together to enable fast, smooth roaming, and a daemon like OpenWrt's
**usteer** adds the active steering intelligence on top that decides *when* to actually trigger one.

## How it works

### The standards

| Standard | Name | What it does |
|---|---|---|
| **802.11k** | Radio Resource Management | AP tells clients which neighboring APs exist and how strong they are |
| **802.11r** | Fast BSS Transition (FT) | Reduces the roaming handshake from ~200 ms to ~1 ms by pre-authenticating with the new AP |
| **802.11v** | BSS Transition Management | Lets the AP *request* a client to roam to a better AP |

**802.11k — Radio Resource Management.** Without it, a client must scan all channels to discover
neighboring APs, which takes time and disrupts traffic. With 802.11k, the AP provides a **Neighbor
Report** on request, so clients can make informed roaming decisions without a blind scan.

**802.11r — Fast BSS Transition.** A normal WPA2 roaming handshake takes 200–500 ms because the
client fully re-authenticates with the new AP from scratch. 802.11r reduces this to under 1 ms by
**pre-exchanging keys with the target AP** before disconnecting from the current one. Two modes
exist: **FT over Air** (the client talks directly to the target AP — more compatible, the usual
choice for a home/SMB setup without a central controller) and **FT over DS** (re-keying traffic
travels over the wired backhaul instead — needs controller infrastructure).

**802.11v — BSS Transition Management.** Without it, an AP can only wait for the client to decide
to roam on its own. 802.11v lets the AP send a **BSS Transition Management Request** — a hint that
the client should move to a specific target AP. Most modern clients honor these requests, though
compliance is client-dependent and not guaranteed.

### usteer — active steering daemon

The 802.11r/k/v parameters alone don't perform any active steering — they only make roaming
*possible* and fast once a client decides to move. **usteer** is an OpenWrt daemon that monitors
all connected clients' signal strength across the AP fleet and sends 802.11v steering requests
automatically — the intelligence layer that decides *when* to trigger a roam, using the mechanism
802.11v provides. It triggers a roam when a client's SNR drops below a configurable threshold, or
when a significantly better AP becomes available (by a configurable dBm margin).

Key `usteer` options (`/etc/config/usteer`):

| Option | Typical value | Meaning |
|---|---|---|
| `signal_diff_threshold` | ~10 dBm | Steer only if the better AP is meaningfully stronger |
| `min_snr` | ~-80 dBm | Minimum acceptable SNR before considering action |
| `roam_scan_snr` | ~-70 dBm | Trigger a neighbor scan when SNR drops below this |
| `roam_kick_snr` | ~-75 dBm | Actively send a 802.11v request when SNR drops below this |
| `load_kick_enabled` | off | Whether to steer clients away from overloaded APs |

```sh
uci show usteer
uci set usteer.@usteer[0].signal_diff_threshold='8'
uci commit usteer
/etc/init.d/usteer restart
```

### OpenWrt configuration

All three standards are configured per SSID in the `wireless` UCI config on each AP that
participates in roaming:

| UCI option | Value | Meaning |
|---|---|---|
| `ieee80211r` | `1` | Enable Fast BSS Transition |
| `ft_psk_generate_local` | `1` | Each AP generates FT keys independently — no auth server needed |
| `mobility_domain` | *(4 hex chars)* | Shared domain ID — must be identical on every AP in the roaming domain |
| `ft_over_ds` | `0` | FT over Air (recommended for home/SMB setups) |
| `ieee80211k` | `1` | Enable Neighbor Reports |
| `ieee80211v` | `1` | Enable BSS Transition Management |

The **Mobility Domain** ties all APs together into a single fast-transition domain — every AP that
participates in seamless roaming for a given SSID must use the same value.

```sh
uci set wireless.<iface>.ieee80211r='1'
uci set wireless.<iface>.ft_psk_generate_local='1'
uci set wireless.<iface>.mobility_domain='<4-hex-id>'
uci set wireless.<iface>.ft_over_ds='0'
uci set wireless.<iface>.ieee80211k='1'
uci set wireless.<iface>.ieee80211v='1'
uci commit wireless
wifi reload
```

### Verification

```sh
# Confirm 802.11r is active on an AP
uci show wireless | grep ieee80211r

# Confirm usteer is running
/etc/init.d/usteer status

# Confirm usteer is seeing clients
ubus call usteer dump_clients

# Watch roaming events in real time
logread -f | grep -i "usteer\|ft\|roam"
```

From a client device, a Wi-Fi analyzer app that shows the associated BSSID (MAC address) confirms
whether the device actually switches to the nearest AP as you move, rather than holding onto a
distant one.

### Rollback

To remove the roaming config from an AP and return it to standard (non-roaming) behavior:

```sh
IFACE=wireless.<iface>
uci delete ${IFACE}.ieee80211r
uci delete ${IFACE}.ft_psk_generate_local
uci delete ${IFACE}.mobility_domain
uci delete ${IFACE}.ft_over_ds
uci delete ${IFACE}.ieee80211k
uci delete ${IFACE}.ieee80211v
uci commit wireless
wifi reload
/etc/init.d/usteer stop
/etc/init.d/usteer disable
```

## When to use it / trade-offs

- Worth enabling as soon as more than one AP shares an SSID — even two APs benefit from clients
  roaming promptly rather than sticking to a fading signal.
- 802.11r client compatibility is generally good today, but scoping it to the SSID/band where
  roaming matters most (typically the highest-traffic, shortest-range band) rather than every
  network reduces the blast radius of any client-compatibility edge case.
- FT over Air (not over DS) is the pragmatic default without dedicated wireless controller
  infrastructure — see [OpenWrt multi-AP fleet policy](../access-points/openwrt-ap-fleet-policy.md)
  for how this fits into a broader fleet-wide policy alongside channel planning and SSID
  consistency.

## Pitfalls

- **A missing steering daemon looks configured.** The 802.11r/k/v UCI options can be entirely
  correct while `usteer` isn't installed, or has silently stopped — always verify the *process* is
  running (`/etc/init.d/usteer status`), not just the wireless config.
- **`mobility_domain` mismatch breaks fast transition silently for that SSID** — every participating
  AP must use the exact same 4-hex-digit value; a typo on one AP doesn't error, it just falls back
  to a slow full re-authentication for clients roaming to/from that AP.
- **SSID string drift undermines roaming.** 802.11r roaming and general client stickiness depend on
  the SSID string matching exactly across every AP — a stray space or case difference silently
  breaks roaming for that network, or makes it look like two separate networks to clients.

## Related

- [OpenWrt multi-AP fleet policy](../access-points/openwrt-ap-fleet-policy.md) — how this
  roaming pattern fits alongside channel planning, IPv6 policy, and SSID/VLAN layout in a fleet of
  dumb APs.

## Sources

- [Legacy wiki.js: OpenWrt Wi-Fi seamless roaming (802.11r/k/v + usteer)](../../../../../sources/network/wifi/2026-09-28-openwrt-seamless-roaming.md)
