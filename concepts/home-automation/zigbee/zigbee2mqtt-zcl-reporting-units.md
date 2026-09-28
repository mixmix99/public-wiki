---
type: concept
title: 'Zigbee2MQTT ZCL reporting: raw units and reducing reporting noise'
description: Why a device's ZCL reporting threshold is always in raw, per-device-scaled units, and how a device can spam full-state republishes via unrelated clusters updating last_seen.
tags: [zigbee2mqtt, zigbee, zcl, mqtt, home-automation]
status: draft
resource:
created: 2026-09-28T17:02:18Z
updated: 2026-09-28T17:02:18Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:02:18Z
verified: []
stale_after: 2028-09-27T17:02:18Z
sources:
- id: shelly-power-strip-4gen4-zigbee-reporting
  resource: 'private:/sources/home-automation/devices/2026-09-28-shelly-power-strip-4gen4-zigbee-reporting.md'
relations: []
superseded_by:
---

# Zigbee2MQTT ZCL reporting: raw units and reducing reporting noise

Two related but distinct problems show up when tuning how often a Zigbee device reports its
attributes to [Zigbee2MQTT](https://www.zigbee2mqtt.io/) (z2m): getting the reporting **threshold
value itself wrong** (silently, with no error), and a device that is **noisy for a reason
unrelated to the attribute you're tuning**.

## How it works

### Reporting thresholds are in raw ZCL units, not human units

A Zigbee Cluster Library (ZCL) attribute like `activePower` or `rmsVoltage` is transmitted as a
raw integer. Home Assistant/z2m convert that integer to a human unit (W, V, A, Hz) using
per-device `divisor`/`multiplier` attributes that the device itself reports over Zigbee. **The
`reportable_change` field of z2m's `bridge/request/device/configure_reporting` MQTT API is in the
same raw units as the attribute — it is never pre-scaled to the human unit you see in the
dashboard.** Get this wrong and the real hysteresis band can be 10–100x off from what was
intended, with no error or warning anywhere in z2m or Home Assistant.

Two attributes are easy to get wrong in opposite directions: a device with `activePower` divisor
`1` needs **no scaling at all** (a raw `5` really is 5 W), while a device with `rmsVoltage`
divisor `100` needs **division by 100**, not 1000 — i.e. centivolts, not millivolts, despite most
people's instinct to assume millivolts given how many other Zigbee devices report voltage. Always
read the device's own `divisor`/`multiplier` (from z2m's cached device database) before setting a
threshold, rather than assuming a "standard" scale.

### A device can be noisy for reasons that have nothing to do with the attribute you suspect

z2m republishes a device's **entire cached state** whenever anything about it changes — including
its tracked `last_seen` timestamp. If a device sends frequent low-level frames on a cluster you
aren't even monitoring (e.g. a vendor-proprietary diagnostic/heartbeat cluster left over from a
different provisioning flow), every one of those frames bumps `last_seen` and triggers a full
republish of every channel and attribute, even though nothing you care about actually changed.

Tightening the reporting configuration for the cluster you *think* is noisy can therefore have
**zero measurable effect** — the real fix is elsewhere. Confirm this by comparing the raw MQTT
payloads across consecutive publishes: if the values you expect to be changing stay byte-for-byte
identical while messages keep arriving, the noise source is a different cluster updating
`last_seen`, not the one you're tuning.

### Debounce vs. reporting thresholds

z2m's per-device `bridge/request/device/options` `{"debounce": <seconds>}` setting coalesces any
updates arriving within that window into a single MQTT publish, regardless of which cluster
triggered them. This is the right tool for "device republishes too often for unrelated reasons" —
tightening ZCL reporting thresholds only helps when the *monitored* attribute itself is changing
too often. Debounce changes require a z2m bridge restart to take effect (a brief interruption of
the whole mesh on that bridge, not just the one device); reporting-threshold changes
(`configure_reporting`) apply live, no restart needed.

## When to use it / trade-offs

- **Tighten `configure_reporting`** when the attribute itself genuinely changes too often for your
  needs (e.g. a very sensitive power sensor reporting every fraction of a watt).
- **Add a per-device `debounce`** when the message rate is high but the actual reported values
  barely change — a strong signal that some other cluster is forcing full-state republishes.
- Keep any threshold that a dependent automation relies on (e.g. detecting a device crossing a
  power draw threshold) fast enough that the automation still sees the transition in time — a
  wider hysteresis band trades responsiveness for fewer messages.

## Pitfalls

- **Always confirm a device's own `divisor`/`multiplier` before setting any `reportable_change`
  value** — never assume a "standard" scale across devices, even for the same physical quantity
  (e.g. voltage in centivolts vs. millivolts on different devices).
- **A silently-wrong threshold has no error message.** The only way to catch it is checking the
  live `configuredReportings` against the device's own divisor/multiplier, or noticing that a
  dependent detection (e.g. "device settled to a low, stable draw") takes far longer to trigger
  than expected because the real hysteresis band is much wider than intended.
- **A loose reporting threshold can mask an unrelated firmware bug for a long time.** If a device
  has a known "value gets stuck" bug, a too-wide threshold means real values can drift for a long
  time without ever crossing the report-triggering delta — giving the bug much more opportunity to
  get stuck without a periodic report to unstick it. Don't rely solely on one metered channel for
  detecting device state if the underlying firmware is known to freeze; use a channel that's
  confirmed to keep updating, or an independent measurement, alongside a timeout-based safety net.
- **When fixing a scaling mistake across multiple identical endpoints of one device, verify every
  endpoint individually** — a correction applied to only some endpoints (because they were
  addressed separately) leaves the rest silently wrong.

## Related

- [Detecting Zigbee2MQTT coordinator hangs](zigbee2mqtt-coordinator-hang-detection.md) — a
  different class of Zigbee2MQTT health-signal pitfall (bridge connection state vs. actual mesh
  health).
- [Setting up SMLIGHT SLZB Zigbee coordinators with Zigbee2MQTT over
  TCP](../../guides/home-automation/zigbee/slzb-zigbee2mqtt-tcp-setup.md) — coordinator-level setup
  this device-level tuning builds on.

## Sources

- [Private source: Shelly Power Strip 4 Gen4 — Zigbee, Home Assistant & Reporting Noise](../../../../../sources/home-automation/devices/2026-09-28-shelly-power-strip-4gen4-zigbee-reporting.md) — private source (real incident)
