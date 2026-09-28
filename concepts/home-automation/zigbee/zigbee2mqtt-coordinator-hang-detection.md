---
type: concept
title: Detecting Zigbee2MQTT coordinator hangs
description: Why the bridge connection state misses radio hangs and how mains-powered canary router devices detect them reliably.
tags:
- zigbee2mqtt
- zigbee
- watchdog
- homeassistant
- smlight
status: draft
resource:
created: 2026-09-27T19:34:18Z
updated: 2026-09-28T17:06:39Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:06:39Z
verified: []
stale_after: 2028-09-26T19:34:18Z
sources:
- id: 2026-09-27-zigbee2mqtt-canary-watchdog
  resource: private:/sources/home-automation/zigbee/2026-09-27-zigbee2mqtt-canary-watchdog.md
- id: zigbee2mqtt-issue-31092
  resource: https://github.com/Koenkk/zigbee2mqtt/issues/31092
- id: slzb-os-scripts-issue-5
  resource: https://github.com/smlight-tech/slzb-os-scripts/issues/5
relations: []
superseded_by:
---

# Detecting Zigbee2MQTT coordinator hangs

A Zigbee coordinator can **hang at the radio level** while Zigbee2MQTT (z2m) still reports the
bridge as connected. A watchdog that should restart the coordinator automatically therefore needs
a signal that reflects whether the **mesh** actually works, not whether the socket is open.

## How it works

z2m exposes two kinds of health signals in Home Assistant:

| Signal | What it really measures | During a radio hang |
|---|---|---|
| Bridge connection state (`binary_sensor.<bridge>_connection_state`) | The MQTT / serial-socket link between z2m and the coordinator | Stays **connected**. The socket is fine, only the radio is dead |
| Device availability (`availability: enabled: true` in z2m) | Whether each device answers z2m's periodic pings | Devices go **`unavailable`** after the ping timeout |

A known example is the hardware bug in SMLIGHT coordinators with the CC2652P7 radio (e.g.
SLZB-MR1). The radio stops handling Zigbee traffic, and z2m logs repeated

```
warning: z2m: Failed to ping '<device>' (... failed (SRSP - AF - dataRequest after 6000ms))
```

for hours. The connection-state sensor typically only blips during the eventual manual restart.

### Canary devices

Pick two or more **mains-powered router devices** (smart plugs, in-wall switch modules) that
never sleep and are spread across the mesh. They answer every availability ping while the mesh
is healthy, so a canary going `unavailable` means something is wrong.

Watchdog pattern (Home Assistant automation):

```yaml
triggers:
  - trigger: state
    entity_id: switch.<canary_1>
    to: unavailable
    for: "00:03:00"
  - trigger: state
    entity_id: switch.<canary_2>
    to: unavailable
    for: "00:03:00"
conditions:
  - condition: and
    conditions:
      - condition: state
        entity_id: switch.<canary_1>
        state: unavailable
      - condition: state
        entity_id: switch.<canary_2>
        state: unavailable
actions:
  - action: script.<restart_coordinator>   # e.g. via the coordinator's HTTP API
```

Either canary triggers the automation. The **AND** condition makes sure it only restarts when both
are down, which means a mesh-wide failure rather than one device losing power.

## When to use it / trade-offs

- **Canary availability:** needs no z2m config change and works with any coordinator. Detection
  lags by the availability timeout plus the `for:` delay.
- **z2m log over MQTT** (`advanced.log_output` including `mqtt`): publishes log lines to
  `<base_topic>/bridge/logging`, so you can trigger on the first `Failed to ping` warning. This is
  earlier and closer to the root cause, but it means parsing log text and changing z2m's config.
- **Connection state:** only useful for detecting a dead z2m process or a broken network or serial
  link to the coordinator, not radio hangs.

## Pitfalls

- **Do not use battery or end devices as canaries.** They sleep, and z2m availability for them is
  based on much longer timeouts. They cause false positives or react far too late.
- A single canary restarts the coordinator whenever that one device is unplugged. Use at least two.
- Check `last_triggered` on the automation after a real incident. A watchdog that has never fired
  may simply be watching the wrong entity.

## Related

- [Setting up SMLIGHT SLZB Zigbee coordinators with Zigbee2MQTT over TCP](../../../guides/home-automation/zigbee/slzb-zigbee2mqtt-tcp-setup.md) — coordinator/bridge setup this watchdog pattern runs against.
- [Zigbee2MQTT ZCL reporting: raw units and reducing reporting noise](zigbee2mqtt-zcl-reporting-units.md) — a different class of Zigbee2MQTT health-signal pitfall (per-device reporting vs. bridge/mesh health).

## Sources

- [Legacy wiki.js: Zigbee2MQTT bridge watchdog canary detection](../../../../../sources/home-automation/zigbee/2026-09-27-zigbee2mqtt-canary-watchdog.md) — private source (real incident)
- [Koenkk/zigbee2mqtt#31092](https://github.com/Koenkk/zigbee2mqtt/issues/31092)
- [smlight-tech/slzb-os-scripts#5](https://github.com/smlight-tech/slzb-os-scripts/issues/5)
