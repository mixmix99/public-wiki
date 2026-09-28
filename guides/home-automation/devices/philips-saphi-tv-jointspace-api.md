---
type: guide
title: Controlling a Philips Saphi TV via JointSpace and Home Assistant, with WoL fallback
description: Generic guide to the JointSpace local API on Philips Saphi TVs, the Home Assistant philips_js integration, and a Wake-on-LAN device-trigger fallback pattern.
tags: [philips, tv, saphi, jointspace, home-assistant, wake-on-lan, api]
status: draft
resource:
created: 2026-09-28T17:02:17Z
updated: 2026-09-28T17:02:17Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:02:17Z
verified: []
stale_after: 2027-09-28T17:02:17Z
sources:
- id: philips-saphi-tv-jointspace-api
  resource: 'private:/sources/home-automation/devices/2026-09-28-philips-saphi-tv-jointspace-api.md'
relations: []
superseded_by:
---

# Controlling a Philips Saphi TV via JointSpace and Home Assistant, with WoL fallback

Philips TVs running the **Saphi** OS (Philips' own Linux-based smart-TV platform, distinct from
the Android TV variant some other Philips models use) expose a local JSON HTTP API called
**JointSpace**. This guide covers the API basics, the official Home Assistant integration, and a
reliable Wake-on-LAN (WoL) fallback for turning the TV on when the API itself is unreachable.

## Prerequisites

- The TV's IP address (and, for the WoL fallback, its MAC address — from your router's DHCP
  lease/reservation table).
- Home Assistant, for the integration steps below (the raw API works standalone too).

## How the JointSpace API works

- **Base URL:** `http://<tv-ip>:1925/6/` (plain HTTP, port 1925, API version 6.1.0 on this
  generation). Some Android-based Philips TVs instead use HTTPS on port 1926 — check which variant
  you have before assuming the URL scheme.
- **Pairing:** Saphi models typically report `pairing_type: none` — no PIN-pairing step, unlike
  Android-based Philips TVs which do require pairing.
- Quick liveness check: `GET http://<tv-ip>:1925/6/system`.

### Endpoints available on Saphi models

| Endpoint | Method | Purpose |
|---|---|---|
| `/6/system` | GET | Device info, API version, feature flags |
| `/6/powerstate` | GET | Current power state (`On` / `Standby`) |
| `/6/input/key` | POST `{"key":"<Key>"}` | Send a remote-control key press |
| `/6/audio/volume` | GET / POST | Volume level and mute state |
| `/6/ambilight/power` | GET | Ambilight on/off |
| `/6/ambilight/currentconfiguration` | GET / POST | Ambilight style |
| `/6/ambilight/mode` | GET | Ambilight mode |
| `/6/ambilight/topology` | GET | LED layout (useful for Ambilight+Hue sync) |
| `/6/activities/tv` | GET | Current channel/activity |

**Not available on Saphi** (403/404 — Android-only features on this API generation):
`/6/sources`, `/6/sources/current`, `/6/context`, `/6/applications`, `/6/recordings/list` (the
last may need a USB tuner/HDD attached even if `/6/system` claims recording support).

### Remote-key emulation

Since Saphi has no direct "set input source" call, most interactive control — including HDMI
switching — goes through remote-key emulation:

```json
POST /6/input/key
Content-Type: application/json

{"key":"Standby"}
```

Known keys: `Standby`, `VolumeUp`, `VolumeDown`, `Mute`, `Source`, `Home`, `Back`, `Options`,
`Info`, `CursorUp`/`CursorDown`/`CursorLeft`/`CursorRight`, `Confirm`,
`RedColour`/`GreenColour`/`YellowColour`/`BlueColour`, `ChannelStepUp`/`ChannelStepDown`,
`PlayPause`, `Play`, `Pause`, `FastForward`, `Rewind`, `Record`, `Subtitle`, `Teletext`.

For HDMI switching: send `Source`, then navigate the on-screen menu with
`CursorUp`/`CursorDown` + `Confirm`. This is slow and depends on the exact on-screen layout —
there is no single-call "switch to HDMI2" endpoint on Saphi.

**`Standby` is not a toggle at the API level.** Sending it while the TV is already off does not
turn it back on — it is a one-way "go to standby" command, unlike the physical remote's power
button (a hardware toggle).

## Power-on: the API alone isn't enough — use Wake-on-LAN

The JointSpace API is only reachable while the TV's system is running. Many Saphi models keep
their network interface alive during standby ("network standby" / quick-start), so the API often
stays reachable even with the screen off — but once the TV is fully asleep, a standard
**Wake-on-LAN magic packet** to its MAC address reliably powers it back on, even when the API is
unreachable:

- **Power off:** `POST /6/input/key {"key":"Standby"}` (works whenever the TV is on).
- **Power on:** WoL magic packet to the TV's MAC (works regardless of API reachability).

If your setup sends the WoL magic packet from inside a container on a Docker bridge network, see
[Wake-on-LAN from a container on a Docker bridge
network](../../../concepts/home-automation/homeassistant/docker-bridge-wake-on-lan.md) — a bridge
network's NAT boundary silently swallows the broadcast.

## Steps

### 1. Add the official Philips TV integration

Home Assistant → Settings → Devices & Services → Add Integration → **Philips TV**
(`philips_js`). Enter the TV's IP address. If `pairing_type` is `none`, setup completes
immediately with no PIN screen. This creates a `media_player` entity (power, volume, source info
where available), plus `light` (Ambilight), `remote` (key emulation), and diagnostic
`switch`/`sensor` entities.

### 2. Wire in a Wake-on-LAN fallback

By default `media_player.turn_on` only succeeds if the JointSpace API is currently reachable — it
calls the API directly. If the TV's network is ever fully down, that call silently does nothing.

**a) Create a `wake_on_lan` switch:**

```yaml
switch:
  - platform: wake_on_lan
    name: "<TV name> Wake on LAN"
    mac: "AA:BB:CC:DD:EE:FF"          # the TV's MAC address
    broadcast_address: "192.0.2.255"  # your subnet's broadcast address
    host: "192.0.2.10"                # optional: TV's IP, enables ping-based state
```

**b) Create an automation on the integration's device trigger.** `philips_js` exposes a device
automation trigger — "device requested to turn on" — that fires whenever the API-direct power-on
path fails. This uses the **older device-trigger schema** (`platform: device`), not the newer
`trigger: <domain>.<type>` shorthand:

```yaml
triggers:
  - trigger: device
    domain: philips_js
    device_id: <philips_js media_player's device_id>   # Settings → Devices, or entity registry
    type: turn_on
conditions: []
actions:
  - action: switch.turn_on
    target:
      entity_id: switch.<your_wol_switch_entity_id>
mode: single
```

Find `device_id` via Settings → Devices & Services → Devices → (your Philips TV device) — it's in
the device detail page's URL, or via the entity registry for the `media_player` entity.

With this in place, `media_player.turn_on` is a single reliable call: if the API is reachable, the
integration turns the TV on directly; if not, this trigger fires and sends the WoL packet instead.
Nothing else needs to know or care which path was actually used.

## Verify

- With the TV fully powered off (network standby lapsed, JointSpace unreachable), call
  `media_player.turn_on` and confirm the device-trigger automation fires and the TV powers on.
- Confirm `powerstate` reports `On` once booted, and that a real power-draw measurement (if you
  have one) rules out a false positive from a device that merely keeps its network interface
  alive.

## Troubleshooting

- **No native HDMI/source switching in Home Assistant for Saphi models** — only remote-key
  emulation is available; the integration doesn't expose a clean `select_source` service for this
  platform.
- **The JointSpace HTTP server can be flaky under rapid/successive requests** — it's a lightweight
  embedded server, not built for fast polling. Space out direct API calls if scripting against it
  outside Home Assistant's own (slower) polling.
- **A device-trigger automation can fail to load right at HA boot** with an error like
  `Integration '<domain>' does not provide trigger support` — this can be a one-time startup race
  rather than a structural problem; reloading the automation domain (or restarting HA core) can be
  enough to fix it. Confirm via HA's own device-trigger listing that the trigger is recognized as
  valid before assuming it's broken.
- **A `wake_on_lan` switch's magic packet reaching nothing** despite correct MAC/broadcast address
  usually means the sender is inside a container on a bridge network — see the linked concept
  above for the fix (host networking, macvlan, or a host-networked relay service).
- **Don't assume plain ICMP ping is a valid "is it on" check** for a TV that keeps its network
  interface alive during standby — it will answer ping even while effectively off. Use the
  JointSpace `powerstate` endpoint and match the literal `"On"` string instead.

## Related

- [Wake-on-LAN from a container on a Docker bridge network](../../../concepts/home-automation/homeassistant/docker-bridge-wake-on-lan.md) — why a bridge-networked container's WoL packet never reaches the LAN, and the fixes.

## Sources

- [Private source: Philips 55OLED754 (2019, Saphi) API & Home Assistant](../../../../../sources/home-automation/devices/2026-09-28-philips-saphi-tv-jointspace-api.md) — private source (real deployment)
