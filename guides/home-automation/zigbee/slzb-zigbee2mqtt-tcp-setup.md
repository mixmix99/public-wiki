---
type: guide
title: Setting up SMLIGHT SLZB Zigbee coordinators with Zigbee2MQTT over TCP
description: Generic setup, adapter selection, and troubleshooting guide for SMLIGHT SLZB Zigbee coordinators connected to Zigbee2MQTT containers over a TCP socket.
tags: [zigbee, slzb, zigbee2mqtt, mqtt, home-automation, docker]
status: draft
resource:
created: 2026-09-28T17:02:16Z
updated: 2026-09-28T17:02:16Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:02:16Z
verified: []
stale_after: 2027-09-28T17:02:16Z
sources:
- id: slzb-zigbee2mqtt-setup
  resource: 'private:/sources/home-automation/zigbee/2026-09-28-slzb-zigbee2mqtt-setup.md'
relations: []
superseded_by:
---

# Setting up SMLIGHT SLZB Zigbee coordinators with Zigbee2MQTT over TCP

An SMLIGHT SLZB Zigbee coordinator (SLZB-06, SLZB-06M, SLZB-06P7, SLZB-MR1, SLZB-07, ...) connects
to [Zigbee2MQTT](https://www.zigbee2mqtt.io/) (z2m) over a **TCP socket** instead of a USB serial
port — useful when the coordinator sits on the network rather than plugged directly into the z2m
host, or when you want to run several independent Zigbee networks/channels from one Docker host,
each with its own coordinator and z2m instance.

## Prerequisites

- One SLZB coordinator per Zigbee network/channel, each with a **static IP** (DHCP changes
  disconnect z2m).
- A Docker (or other container) host able to reach each coordinator's IP over the network.
- An MQTT broker reachable from the z2m host.

## Architecture

```
Zigbee Devices
     │ (802.15.4)
     ▼
SLZB Coordinator (Ethernet, static IP)
     │ (TCP socket, typically port 6638 or 7638)
     ▼
Zigbee2MQTT container
     │ (MQTT)
     ▼
MQTT Broker
     │
     ▼
Home automation platform
```

Each coordinator serves one Zigbee network/channel and needs its own z2m container instance with
a unique MQTT base topic — don't try to share one z2m instance across multiple coordinators.

## Steps

### 1. Identify the correct adapter type

The z2m `adapter:` setting depends on the **Zigbee radio chip** inside the specific SLZB model,
not the product line as a whole:

| Zigbee chip | Adapter |
|---|---|
| Texas Instruments CC2652P / CC2652P7 | `zstack` |
| Silicon Labs EFR32 | `ember` |

Find the right value for your unit either from the SLZB web UI's own generated z2m config string
(ZHUB Config page), or from the coordinator's `/config` API: if `coordMode` is present and the
reported firmware date is after `2023-10-30`, use `ember`.

### 2. Configure the SLZB's socket connection settings

In the SLZB web UI, under **ADVANCED → Socket connection options**:

| Setting | Recommended | Why |
|---|---|---|
| Enable Zigbee Socket packet processing | ON | Adds packet framing/validation, preventing malformed packets from crashing the TCP socket |
| Allow multi-threaded socket connection | OFF | Only one client (one z2m instance) should ever connect to a given coordinator's socket. Leaving this on lets a second client (monitoring, a port scan, a second z2m instance) cause `ECONNRESET` crash loops |
| Multi-Radio Queue Control | Default | Only relevant if the same SLZB also runs Thread/BLE alongside Zigbee |

Also: keep the coordinator's firmware up to date (firmware bugs can cause socket instability), and
avoid enclosed placements — these devices can overheat and destabilize.

### 3. Configure the Zigbee2MQTT container

Key settings in the instance's `configuration.yaml`:

```yaml
serial:
  port: tcp://<coordinator-ip>:<port>   # e.g. 6638 or 7638
  baudrate: 115200
  adapter: zstack        # or 'ember' — see step 1
  disable_led: false

mqtt:
  base_topic: zigbee2mqtt/<instance-name>
  server: mqtt://<broker-ip>
  user: <mqtt-user>
  password: <mqtt-password>
  keepalive: 60
  version: 4

advanced:
  transmit_power: 20     # max power
  channel: <channel>     # unique per coordinator to avoid interference
  log_level: info
  last_seen: ISO_8601_local

frontend:
  enabled: true
  port: 8080             # internal port, map to a host port per instance

availability:
  enabled: true
```

Give each instance its own compose stack/container, data directory, and MQTT base topic. Running
multiple coordinators as separate containers (rather than one z2m instance managing multiple
serial ports) keeps each Zigbee network's failure domain isolated.

## Verify

- `nc -zv <coordinator-ip> <port>` succeeds.
- The z2m container's logs show a successful adapter start with no repeated `ECONNRESET`.
- The instance's web frontend lists paired devices and shows them receiving updates.

## Troubleshooting

### `ECONNRESET` / adapter disconnected / restart loops

Symptom (in the z2m container logs):

```
zh:zstack:znp: Socket error Error: read ECONNRESET
zh:zstack:znp: Port closed
z2m: Adapter disconnected, stopping
```

Check causes in this order (most likely first):

1. Multi-threaded socket enabled on the SLZB (step 2) — disable it.
2. SLZB firmware bug — update firmware via its web UI.
3. Network instability — ping the coordinator's IP, check for packet loss.
4. Coordinator overheating — move it to a ventilated location.
5. A second client is connected to the socket — check open connections to the coordinator's port
   from the z2m host.

### A container restart briefly disrupts every other container on the same bridge network

If several containers (multiple z2m instances plus other services) share one Docker bridge
network, restarting any one of them briefly drops connectivity for all the others. This is not
Zigbee-specific — check the bridge's Spanning Tree forward-delay setting: a non-zero forward delay
(the Linux bridge default is 15s) makes a routine veth recreate on container restart cause several
seconds of connectivity loss for the whole bridge. Setting the bridge's forward delay to 0 (when
STP itself is not needed) removes this disruption, though the setting typically needs to be
reapplied on every host boot (e.g. via a startup script or `@reboot` cron entry).

### "Failed to ping" warnings

```
Failed to ping 'DeviceName' (Timeout after 10000ms)
```

Normal for battery-powered sleepy devices or devices at the edge of the mesh. Not evidence of a
coordinator problem by itself — only worth investigating further if it affects mains-powered
devices or routers, which should always respond.

## Related

- [Detecting Zigbee2MQTT coordinator hangs](../../../concepts/home-automation/zigbee/zigbee2mqtt-coordinator-hang-detection.md) — the connection-state sensor can stay "connected" while the coordinator radio itself is hung; use mains-powered canary devices instead.
- [Zigbee2MQTT ZCL reporting: raw units and reducing reporting noise](../../../concepts/home-automation/zigbee/zigbee2mqtt-zcl-reporting-units.md) — a separate class of per-device tuning once devices are joined.

## Sources

- [Private source: SLZB Coordinators & Zigbee2MQTT Setup](../../../../../sources/home-automation/zigbee/2026-09-28-slzb-zigbee2mqtt-setup.md) — private source (real deployment configuration)
