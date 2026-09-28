---
type: concept
title: SMART health polling can wake drives despite an idle spindown timer
description: Why a periodic SMART health check that ignores drive power state can repeatedly wake spun-down drives even when a separate idle-timeout spindown tool is working correctly.
tags: [truenas, smart, spindown, hdd, zfs]
status: draft
resource:
created: 2026-09-28T17:04:24Z
updated: 2026-09-28T17:04:24Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:04:24Z
verified: []
stale_after: 2028-09-27T17:04:24Z
sources:
- id: 2026-09-28-smart-polling-wakes-spundown-drives
  resource: 'private:/sources/administration/truenas/2026-09-28-smart-polling-wakes-spundown-drives.md'
relations: []
superseded_by:
---

# SMART health polling can wake drives despite an idle spindown timer

An idle-timeout spindown tool and a periodic SMART/health-check poll are two independent systems
that both touch the same drives. If the health poll doesn't check power state before running, it
will wake a drive the spindown tool correctly put to sleep — and the spindown tool has no way to
prevent it, because the wake-up doesn't originate from the tool it's designed to coexist with.

## How it works

1. A spindown tool (or a NAS OS's own idle-power feature) puts a drive into standby after N
   seconds of no I/O.
2. Separately, a health-monitoring subsystem runs `smartctl` (or an equivalent SMART-reading
   command) against every disk on a fixed schedule, regardless of its current power state.
3. Most SMART tooling has an explicit standby-aware mode (e.g. `smartd -n standby`) that skips a
   sleeping drive rather than issuing a command that would spin it up. If the specific code path
   running the health check does **not** use that mode — for example, a newer or different
   subsystem than the one the standby-aware flag was written for — it silently wakes the drive.
4. From the outside this looks like the spindown tool "not working" (drives keep spinning back
   up), when the actual spindown logic is fine — a completely separate process is undoing it on a
   fixed interval.

## When to use it / trade-offs

This is a diagnostic pattern, not a feature to opt into: any system with more than one component
that can touch drive power state (a spindown daemon, a temperature-graphing agent, a health-alert
subsystem, a backup job) can hit this. The trade-off is inherent to layering: the more independent
subsystems poll hardware directly instead of going through a single power-state-aware abstraction,
the more likely one of them ignores standby state.

## Pitfalls

- **Don't assume the spindown tool is broken just because drives keep waking up.** Check what else
  polls the drives on a schedule — temperature monitoring, health/SMART alerting, backup discovery
  scans — before touching the spindown tool's own logic. See the investigation this concept was
  extracted from: a NAS OS's health-alert subsystem update added a fixed-interval SMART poll with
  no standby check, which looked exactly like a spindown regression until the actual polling code
  was traced (private write-up linked below).
- **A fix applied to a system file inside a read-only/immutable OS layer doesn't survive an
  upgrade.** If the only way to fix the polling code is editing a file that ships with the OS
  itself, expect to reapply the patch after every OS update — and expect the exact code location
  or indentation to shift between versions, breaking a naive automated re-patch script.
- **Restarting the service you just patched can kill an unrelated process you still need.** If the
  spindown tool runs as a child process of the same service you restart to apply the fix, the
  restart silently kills the spindown tool too — a patch can look complete and correct while the
  drives still never spin down, for an entirely different reason than the one just fixed.
## Related

<None yet.>

## Sources

- [Legacy wiki.js: TrueNAS Scale 25.10 SMART polling wakes spun-down drives](../../../../../sources/administration/truenas/2026-09-28-smart-polling-wakes-spundown-drives.md) — private source (original investigation, including the real affected host)
