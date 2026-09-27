---
type: concept
title: Diagnosing asymmetric gigabit link faults with bidirectional iperf3
description: Before blaming Wi-Fi or a chipset for a throughput problem, run iperf3 in both directions between two wired hosts to rule out a one-directional cable/switch-port fault.
tags: [networking, iperf3, troubleshooting, ethernet]
status: draft
resource:
created: 2026-09-27T19:56:24Z
updated: 2026-09-27T19:56:24Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T19:56:24Z
verified: []
stale_after: 2028-09-26T19:56:24Z
sources:
- id: 2026-09-27-ap-fleet-throughput-investigation
  resource: private:/sources/network/wifi/2026-09-27-ap-fleet-throughput-investigation.md
relations: []
superseded_by:
---

# Diagnosing asymmetric gigabit link faults with bidirectional iperf3

When throughput between two hosts is bad in only one direction, the fault is almost always in a
physical cable or switch port, not in Wi-Fi, a driver, or a chipset — even if the symptom first
showed up over Wi-Fi elsewhere on the same network. A single bidirectional `iperf3` test between
two wired hosts is enough to prove or rule this out before chasing wireless-specific causes.

## How it works

1. Pick two hosts on the suspect path that you can both reach directly over wired Ethernet
   (e.g. a server and a switch-adjacent host), even if the original symptom was seen on a
   wireless client further downstream.
2. Start an `iperf3` server on one host: `iperf3 -s`.
3. Run the client test **in both directions**, not just one:
   ```
   iperf3 -c <host-a-ip>          # host B -> host A
   iperf3 -c <host-b-ip>          # host A -> host B (reverse the client/server roles, or use -R)
   ```
4. Compare throughput and retransmit counts in each direction. A clean gigabit result one way
   (near line rate, ~0 retransmits) and a capped, retransmit-heavy result the other way is the
   signature of a **directional fault** — most commonly a marginal cable, a bad pair, or a flaky
   switch port — not a symmetric bandwidth or congestion problem.
5. Cross-check with `ethtool` on both NICs for negotiated link speed/duplex and hardware error
   counters (`ethtool -S <iface>`). If the link negotiates full gigabit duplex and reports **zero**
   hardware TX/RX errors on both ends even during the failing direction, that rules out gross
   signal-integrity or duplex-mismatch causes and narrows the fault further to the cable run or an
   intermediate switch port, which typically won't show up as a NIC-level error counter.
6. Optionally repeat with a UDP test (`iperf3 -u`) to get an explicit loss percentage instead of
   relying on TCP retransmit counts, which can be throttled by the congestion-control algorithm.

## When to use it / trade-offs

- Use this **early**, before investing time in driver-, chipset- or roaming-specific
  investigations — it takes a few minutes and immediately tells you whether the problem is
  wired-infrastructure or something else.
- Requires two hosts that can both run `iperf3` and that you can reach directly (not just through
  the suspect Wi-Fi hop) — if only Wi-Fi clients are available, test AP-to-router or AP-to-server
  over the wired uplink instead of client-to-AP over the air.
- A one-directional result is strong evidence, not absolute proof: rare cases (e.g. a router doing
  asymmetric QoS shaping) can also look directional. Ruling out shaping/QoS on the path is a
  reasonable next step if reseating/replacing the cable doesn't fix it.

## Pitfalls

- **Trying a link-speed downgrade to "fix" it mid-test:** forcing a lower link speed
  (e.g. 100 Mb instead of 1000 Mb) to see if a marginal cable behaves better at a lower rate causes
  a brief link renegotiation/drop. Don't do this on a host that is your only access path into a
  remote network — you may lock yourself out if the link doesn't come back up cleanly.
- **Trusting NIC error counters alone:** a completely clean `ethtool -S` on both ends during a
  failing test does not rule out a physical-layer problem — it only means the *adapters* aren't
  reporting errors. The fault can still be in the cable run or an intermediate switch port between
  them, which won't be visible from either NIC's own counters.
- **Assuming a wireless symptom means a wireless cause:** a throughput problem first noticed on a
  Wi-Fi client can originate entirely on the wired side of the network (e.g. the AP's own uplink
  cable), especially in a network with long in-wall cable runs to APs.

## Related

No linked entities or guides in this wiki yet.

## Sources

- [Legacy wiki.js: internet instability and AP throughput collapse after fleet replacement](../../../../../sources/network/wifi/2026-09-27-ap-fleet-throughput-investigation.md) — private raw source; link works in the combined checkout, dead on the standalone public repo
