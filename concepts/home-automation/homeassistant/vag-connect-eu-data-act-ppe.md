---
type: concept
title: vag_connect EU Data Act fallback on PPE vehicles
description: How the vag_connect integration falls back to the EU Data Act portal, the known empty-feed bug on VW Group PPE vehicles, and why re-importing an export is no workaround.
tags: [homeassistant, vag-connect, eu-data-act, vw-group, audi]
status: draft
resource: https://github.com/its-me-prash/vwgroup-connect-ha
created: 2026-09-27T19:34:18Z
updated: 2026-09-27T19:34:18Z
generated:
  by: claude/opus-5.5
  at: 2026-09-27T19:34:18Z
verified: []
stale_after: 2028-09-26T19:34:18Z
sources:
- id: 2026-09-27-vag-connect-eu-data-act
  resource: private:/sources/home-automation/homeassistant/2026-09-27-vag-connect-eu-data-act.md
- id: vwgroup-connect-ha-issue-1403
  resource: https://github.com/its-me-prash/vwgroup-connect-ha/issues/1403
relations: []
superseded_by:
---

# vag_connect EU Data Act fallback on PPE vehicles

[`vag_connect`](https://github.com/its-me-prash/vwgroup-connect-ha) is a HACS custom integration
that brings VW Group vehicles (VW, Audi, Škoda, …) into Home Assistant. On newer **PPE-platform**
vehicles (e.g. Audi Q6 e-tron, S6/A6 e-tron) it may never receive data, and that is an upstream
problem, not a configuration error.

## How it works

The integration has two data channels:

1. **Native app backend:** logs in like the brand's mobile app. For Audi (myAudi) this headless
   login is blocked and fails reliably.
2. **EU Data Act portal:** a read-only channel based on the EU Data Act. The owner registers the
   vehicle on the manufacturer's portal, grants consent and creates a **continuous data request**.
   The integration then polls the published datasets (15-minute intervals).

When the native login fails, the integration falls back to channel 2.

## Pitfalls

- **Empty continuous feed on PPE vehicles.** The portal channel authenticates fine, but every
  dataset is a `no_content_found.zip` placeholder, and HA keeps saying "no vehicle data yet".
  VW's backend does not populate the continuous feed for PPE vehicles, while a **one-off
  historical export** from the same portal does contain data. This is a known, open upstream bug:
  [vwgroup-connect-ha#1403](https://github.com/its-me-prash/vwgroup-connect-ha/issues/1403).
  Before debugging, confirm ownership, consent and the continuous request on the portal. If they
  are all correct, there is nothing to fix locally.
- **Re-importing the one-off export is not a workaround.** The integration's merge logic only
  fills fields that are currently `null` and never overwrites them. The first import fills
  values, but every later import is a no-op, so a daily export import will not give you current
  data.
- Check the upstream issue for progress before re-diagnosing from scratch.

## Sources

- [Legacy wiki.js: Audi vag_connect EU Data Act troubleshooting](../../../../../sources/home-automation/homeassistant/2026-09-27-vag-connect-eu-data-act.md) — private source
- [vwgroup-connect-ha#1403](https://github.com/its-me-prash/vwgroup-connect-ha/issues/1403)
