---
type: concept
title: 'OpenWrt throughput pitfalls: ath10k key install failures and TCP collapse over Wi-Fi'
description: 'Two separate OpenWrt/Wi-Fi throughput failure patterns: a long-standing ath10k/hostapd 802.11r key-installation bug, CPU-bound bridging on weak single-core APs, and TCP throughput collapse for fast senders crossing a Wi-Fi hop.'
tags: [openwrt, wifi, ath10k, hostapd, throughput, tcp, vlan, bridge]
status: draft
resource:
created: 2026-09-27T19:54:27Z
updated: 2026-09-27T19:54:27Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T19:54:27Z
verified: []
stale_after: 2028-09-26T19:54:27Z
sources:
- id: openwrt-ath10k-throughput-investigation
  resource: private:/sources/network/wifi/2026-09-27-openwrt-ath10k-throughput-investigation.md
- id: wifi-hop-tcp-throughput-collapse
  resource: private:/sources/network/wifi/2026-09-27-wifi-hop-tcp-throughput-collapse.md
- id: openwrt-issue-9071
  resource: https://github.com/openwrt/openwrt/issues/9071
- id: openwrt-issue-6937
  resource: https://github.com/openwrt/openwrt/issues/6937
relations: []
superseded_by:
---

# OpenWrt throughput pitfalls: ath10k key install failures and TCP collapse over Wi-Fi

Three distinct, easily conflated throughput failure patterns show up repeatedly on OpenWrt access points: a long-standing driver-level key-installation bug on `ath10k` hardware, CPU-bound softirq saturation on weak single-core APs when a second bridged network is added, and a TCP-specific throughput collapse for fast senders whose packets cross any Wi-Fi hop. They have different root causes and different fixes — don't assume one explains a symptom that looks similar to another.

## How it works

### `ath10k`/`hostapd` key-installation failures with 802.11r

Some `ath10k`-based APs running 802.11r (fast BSS transition / FT roaming) across multiple virtual APs (VAPs) on a 5GHz radio log recurring errors like:

```
hostapd: phy0-ap0: nl80211: kernel reports: key addition failed
ath10k_pci: SWBA overrun on vdev N, skipped old beacon
```

`key addition failed` means WPA key installation into the hardware crypto engine is failing — sometimes even with **zero clients connected**, i.e. periodically rather than purely client-triggered. `SWBA overrun` means beacon frames are being dropped under load. Both strings have long, still-open histories upstream across multiple chipset vendors and OpenWrt releases going back years — they are not necessarily a fresh regression in whatever version you're currently running, and are frequently (though not always) dismissed by the community as harmless log noise that doesn't affect real throughput. Whether it's harmless or catastrophic on a given device seems to depend on frequency and correlation with actual throughput collapse, not the presence of the message alone.

### CPU-bound bridging on weak single-core APs

On low-power, single-core APs, adding a **second netifd-managed bridge interface** (e.g. for an isolated guest VLAN) can saturate the CPU's softirq (network packet processing) path entirely — independent of whether that bridge carries any traffic. A `top` snapshot during a transfer on affected hardware can show CPU softirq jump from ~20% to **100%** the moment the interface exists, regardless of DHCP/static/L3-less protocol choice on it. The VLAN device and bridge device alone, with no managed `interface` section on top, are typically free — it's specifically the routing/netifd management layer that costs the CPU.

### TCP throughput collapse for fast senders over a Wi-Fi hop

A host capable of line-rate 10G or 2.5G sending TCP through any Wi-Fi hop (even to a Wi-Fi-attached endpoint with plenty of real capacity) can collapse to a small fraction of the link's actual capacity, while a 1G-limited sender crossing the identical Wi-Fi hop performs fine. The discriminating signal is **loss pattern, not loss amount**: the fast sender shows continuous low-level retransmit "drizzle" for the whole transfer even while running at a fraction of the link's measured UDP capacity, with its TCP congestion window capped far below what a slower sender achieves; the slow sender's retransmits are confined to the initial slow-start burst, after which it cruises loss-free. The working theory is that line-rate micro-bursts leaving a fast NIC interact badly with a slower downstream hop (a Wi-Fi radio, or a bandwidth step-down point such as a 10G-to-2.5G switch transition) in a way standard TCP congestion control doesn't compensate for well.

## When to use it / trade-offs

- **Don't assume a key-installation or beacon-overrun error explains a throughput problem** without checking whether it correlates with actual measured throughput collapse and occurs even with zero clients connected. On some hardware it's genuinely benign log spam; on others it correlates with severe, measurable throughput loss.
- **Isolated guest networking on weak single-core AP hardware may simply not be achievable** without a significant throughput cost, if the bottleneck is CPU/softirq rather than anything configurable. Bridging the guest SSID directly into the main LAN (accepting no isolation) is a legitimate trade-off in low-risk deployment locations, if a more capable AP isn't an option.
- **TCP pacing techniques (kernel-level, not application-level) and modern congestion control (e.g. BBR) are the most promising fix directions** for the fast-sender-over-Wi-Fi collapse pattern, since the mechanism points at burst behavior rather than the wireless link's fundamental capacity.

## Pitfalls

- **Verify actual AP association before trusting a throughput number**, especially with roaming (802.11r/k/v) active — a client can silently be associated with a different AP than the one physically closest to it, producing misleadingly bad (or good) numbers that have nothing to do with the AP you're testing.
- **Bisect with a full clean reboot at every step, not just a config reload.** Driver/kernel state from repeated `reload`/`down`/`up` cycles can mask or mimic the real effect of a configuration change, especially for CPU-bound or driver-state bugs.
- **A negative version-bisection result is still informative.** If no version in a bisection range reproduces a bug that was definitely observed in production, the bisected variable (version) is probably not the real trigger — look for a confounding factor (an additional config change, uptime/resource accumulation) that was present in the original failure but not replicated in the bisection's fresh-boot test conditions.
- **Don't conflate "wired path tested once and found clean" with "wired path is generally clean."** A single test between two specific endpoints does not rule out a fault elsewhere in a multi-hop switch chain reachable by other wired ports.
- **Application-level rate limiting (e.g. `iperf3 -b`) is not the same as kernel-level packet pacing** — it throttles the application's send calls but bursts can still leave the NIC at line rate. A tool offering a true kernel-pacing option (e.g. `iperf3 --fq-rate`) is needed to actually test whether burst behavior is the cause.

## Related

- No related public pages yet in this domain.

## Sources

- [Legacy wiki.js (private): OpenWrt AP throughput collapse investigation (ath10k key-installation bug)](../../../../../sources/network/wifi/2026-09-27-openwrt-ath10k-throughput-investigation.md) — private source (real incident)
- [Legacy wiki.js (private): downlink throughput investigation across a Wi-Fi hop](../../../../../sources/network/wifi/2026-09-27-wifi-hop-tcp-throughput-collapse.md) — private source (real incident)
- [OpenWrt GitHub #9071 — key addition failed with 802.11r](https://github.com/openwrt/openwrt/issues/9071)
- [OpenWrt GitHub #6937 — ath10k SWBA overrun, AP drops all clients](https://github.com/openwrt/openwrt/issues/6937)
