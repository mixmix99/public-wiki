---
type: concept
title: OpenWrt multi-AP fleet policy
description: 'Generic policy pattern for a fleet of dumb OpenWrt access points: 802.11r/k/v roaming, a fixed 5GHz / auto 2.4GHz channel plan, disabling IPv6 on dumb APs, and consistent SSID naming.'
tags:
- openwrt
- wifi
- access-point
- roaming
- channel-plan
- ipv6
status: draft
resource:
created: 2026-09-27T19:48:50Z
updated: 2026-09-28T17:07:17Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:07:17Z
verified: []
stale_after: 2028-09-26T19:48:50Z
sources:
- id: ap-fleet-overview
  resource: private:/sources/network/access-points/2026-09-27-ap-fleet-overview.md
relations: []
superseded_by:
---

# OpenWrt multi-AP fleet policy

When several OpenWrt access points cover one building, treating them as independent devices
produces inconsistent Wi-Fi: clients that stick to a weak AP, config drift after firmware updates,
and channels that silently converge onto the same congested block. Running them as a **fleet** with
one written-down policy — applied identically to every AP regardless of hardware — fixes this.

## How it works

Every AP in the fleet runs in **dumb AP mode**: no routing, no DHCP server, just a bridge from
Wi-Fi to the wired LAN, with a central router/firewall handling DHCP and routing. On top of that,
four settings are enforced identically everywhere:

### 1. Seamless roaming (802.11r/k/v + a steering daemon)

Configure Fast BSS Transition (802.11r), Neighbor Reports (802.11k) and BSS Transition Management
(802.11v) on the SSID(s) where roaming matters most (typically the highest-traffic/shortest-range
band). See [OpenWrt seamless Wi-Fi roaming with 802.11r/k/v and usteer](../wifi/openwrt-seamless-roaming.md)
for the full standards explanation, `usteer` internals, verification commands and a rollback
procedure — summarized here as one of the fleet's four policy pillars:

| UCI option | Value | Meaning |
|---|---|---|
| `ieee80211r` | `1` | Enable Fast BSS Transition |
| `ft_psk_generate_local` | `1` | Each AP generates FT keys independently (no need to share a key manually) |
| `mobility_domain` | `<4-hex-digit-id>` | Shared domain ID — must be **identical** on every AP |
| `ft_over_ds` | `0` | FT over the air (not over the distribution system) |
| `ieee80211k` | `1` | Enable Neighbor Reports |
| `ieee80211v` | `1` | Enable BSS Transition Management |

The 802.11r/k/v parameters alone don't perform any active steering — they only make roaming
*possible* and fast once a client decides to move. A steering daemon (OpenWrt's `usteer` is the
common choice) is what actually nudges clients toward a better AP, by watching signal strength
across the fleet and sending 802.11v requests. **`usteer` must be installed and running on every
AP** — a missing or silently-stopped daemon leaves roaming looking "configured" (correct UCI
options) while doing nothing.

Typical `usteer` thresholds:

| Option | Typical value | Meaning |
|---|---|---|
| `signal_diff_threshold` | ~10 dBm | Steer only if a neighboring AP is meaningfully stronger |
| `min_snr` | ~-80 dBm | Minimum acceptable SNR before a client is considered poorly served |
| `roam_scan_snr` | ~-70 dBm | SNR below which a neighbor scan is triggered |
| `roam_kick_snr` | ~-75 dBm | SNR below which an 802.11v transition request is sent |

Reasons to scope roaming to one band (e.g. 5GHz only) rather than applying it everywhere:
- The shorter-range band has more frequent, more impactful roaming events.
- IoT/smart-home devices on other SSIDs avoid any 802.11r client-compatibility edge cases.

### 2. Fixed 5GHz channels, auto 2.4GHz

Run the two bands with **opposite** channel policies:

- **5GHz: fixed, never `auto`.** Plan channels by hand based on which APs can actually hear each
  other (an interference graph, not just physical distance — building materials like metal-
  reinforced floors matter more than raw distance). Fixed channels stop each AP's auto-channel-
  select (ACS) algorithm from independently re-converging onto the same "quiet-looking" channel
  over time, which is what causes multiple APs to silently pile onto one channel months after a
  careful initial rollout.
- **2.4GHz: always `auto`, never fixed.** 2.4GHz has far fewer non-overlapping channels and
  (in a residential/mixed environment) much more external interference from neighboring networks
  and non-Wi-Fi devices; a static assignment gets stale faster than ACS can adapt, so the usual
  logic is inverted here.

**Gotcha:** setting a DFS channel (the UNII-2/UNII-2e ranges, roughly 52-144 depending on
regulatory domain) via UCI **fails outright** if the radio's regulatory `country` code isn't set
first — `hostapd` rejects the channel immediately (`NO-IR RADAR` / "Could not select hw_mode and
channel"), and the radio never starts, rather than just risking the wrong TX power. Always set
`country` **before or together with** a DFS channel:

```sh
uci set wireless.radioN.country="<CC>"
uci set wireless.radioN.channel="<target>"
uci commit wireless
wifi reload radioN
```

A DFS channel change also triggers Channel Availability Check (CAC, commonly ~60 seconds) before
the radio comes up — this is normal, not a failure.

### 3. IPv6 disabled fleet-wide on dumb APs

If the fleet's upstream network doesn't route IPv6, or IPv6 isn't wanted on the Wi-Fi segment at
all, disable it explicitly on every AP rather than leaving it to chance — an AP left at OpenWrt
defaults will happily relay/advertise IPv6 it has no business handling.

```sh
# Disable IPv6 entirely on the LAN bridge device (kernel-level — no RA/DHCPv6/SLAAC possible)
uci set network.@device[<idx-of-lan-bridge>].ipv6='0'

# Remove any IPv6 prefix delegation on the lan interface (UCI section name, not a hostname)
uci -q delete network.lan.ip6assign  # wiki:allow
uci -q delete network.lan.ip6ifaceid  # wiki:allow
uci -q delete network.lan.ip6hint  # wiki:allow

# Belt-and-braces: explicitly disable IPv6 in dnsmasq/odhcpd too
uci set dhcp.lan.ra='disabled'  # wiki:allow
uci set dhcp.lan.dhcpv6='disabled'  # wiki:allow
uci set dhcp.lan.ndp='disabled'  # wiki:allow

uci commit network
uci commit dhcp
/etc/init.d/network restart
```

The device-level `ipv6='0'` on the LAN bridge is the important part — it prevents the kernel from
doing anything IPv6 on that interface at all. The `dhcp` options are redundant given that, but kept
for clarity of intent and defense in depth.

Verify with:
```sh
ip -6 addr show <lan-bridge>   # should show nothing
uci get dhcp.lan.ra            # should print 'disabled'  # wiki:allow
```

### 4. Consistent SSID/VLAN layout

Every AP should broadcast an identical set of "fleet" SSIDs, byte-identical in spelling (802.11r
roaming and general client stickiness depend on the SSID string matching exactly — a stray space or
case difference like `5Ghz` vs `5GHz` silently breaks roaming for that SSID, or creates what looks
like two separate networks). A typical layout:

| SSID pattern | Scope | Notes |
|---|---|---|
| `<Main>` / `<Main> 5GHz` | Every AP | Roaming-enabled band gets the suffix |
| `<Category>` (e.g. home automation) | Every AP | Separate SSID per device category, own VLAN if segmented |
| `<Guest>` | Every AP | See VLAN note below |
| `<Location>` per-AP SSID | One AP | Optional — lets a client pin itself to one physical AP |

If a guest network needs real L2 isolation, model it as its own VLAN with its own subnet. Be aware
this can have a real throughput cost on constrained/single-core hardware — see
[Pitfalls](#pitfalls).

## When to use it / trade-offs

- Worth doing as soon as a "fleet" exists — even three or four APs benefit from a fixed channel
  plan and roaming, and the policy scales to any size without changing shape.
- The fixed-5GHz/auto-2.4GHz split assumes 2.4GHz interference is the dominant problem and 5GHz
  interference is planable; in an environment with heavy 5GHz congestion too (e.g. dense
  apartment buildings), both bands may need manual planning instead.
- 802.11r client compatibility is generally good today, but scoping it to one SSID/band rather than
  every network reduces the blast radius of any edge case.

## Pitfalls

- **A missing steering daemon looks configured.** 802.11r/k/v UCI options can be entirely correct
  while `usteer` (or equivalent) isn't installed or has silently stopped — verify the *process* is
  running, not just the config.
- **SSID string drift breaks roaming silently.** A single AP with a typo'd or differently-cased SSID
  doesn't error — it just creates a client-visible inconsistency (or, worse, looks like a separate
  network entirely) that undermines the whole roaming setup. Audit exact SSID strings across the
  fleet periodically, not just once at rollout.
- **DFS channels fail hard without `country` set** — not a soft warning, the radio doesn't start.
  Set the regulatory domain on every radio before or with any DFS channel assignment.
- **ACS re-converges over time.** A "temporarily fine" auto channel plan on 5GHz will often drift
  back to whatever channel looks quietest to each AP's independent scan, undoing a manual balancing
  effort — this is exactly why 5GHz should be fixed, not just initially set.
- **A second managed network interface (e.g. a VLAN-backed guest bridge) can cost significant
  throughput on single-core/low-end hardware**, independent of whether it's actually passing
  traffic — the mere presence of an extra `netifd`-managed interface was enough to measurably hurt
  throughput on constrained devices in one real fleet. If guest isolation is wanted, budget time to
  benchmark it per hardware class rather than assuming it's free.

## Related

- [OpenWrt seamless Wi-Fi roaming with 802.11r/k/v and usteer](../wifi/openwrt-seamless-roaming.md) — deep dive on this policy's roaming pillar.

## Sources

- [Legacy wiki.js: AP fleet directory, inventory and Wi-Fi policy](../../../../../sources/network/access-points/2026-09-27-ap-fleet-overview.md) — private source (real fleet this pattern was distilled from)
