---
type: concept
title: Pulse self-hosted monitoring
description: How Pulse's server/agent monitoring pattern works, and common false-alarm pitfalls (clock drift, stale snapshot fields, auth-required API).
tags: [monitoring, pulse, self-hosted, docker, observability]
status: draft
resource:
created: 2026-09-28T17:01:15Z
updated: 2026-09-28T17:01:15Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:01:15Z
verified: []
stale_after: 2028-09-27T17:01:15Z
sources:
- id: 2026-09-28-pulse-monitoring-pitfalls
  resource: 'private:/sources/administration/services/2026-09-28-pulse-monitoring-pitfalls.md'
relations: []
superseded_by:
---

# Pulse self-hosted monitoring

[Pulse](https://github.com/rcourtman/pulse) is a self-hosted monitoring dashboard: a central
**Pulse server** (a Docker container) collects data pushed by lightweight **agents** running on
each monitored host, and presents host metrics, Docker container status, alerts, and historical
graphs in a web UI.

## How it works

- **Server**: a single Docker container exposing a web UI (default port `7655`). All state
  (configuration, metrics database, alert history, encrypted credentials) lives under a data
  directory mounted into the container.
- **Agent** (`pulse-agent`): a small Go binary installed on every monitored host as a systemd
  service (`pulse-agent.service`). It periodically collects local data and pushes it to the server
  over HTTPS using a per-host API token.

Typical agent invocation:

```
pulse-agent --url https://<pulse-server-url> \
  --interval 30s \
  --token-file /var/lib/pulse-agent/token \
  --enable-host \
  --enable-docker \
  --state-dir /var/lib/pulse-agent
```

- `--enable-host` reports system metrics: CPU usage, memory, load average, disks, disk I/O,
  network interfaces, sensors, uptime.
- `--enable-docker` reports Docker container status and health-check results (e.g. raises an alert
  when a container's healthcheck transitions to `unhealthy`).
- `--enable-kubernetes` / `--enable-commands` exist for Kubernetes monitoring and remote command
  execution, disabled by default.

The agent buffers reports locally (persisted to disk) if it can't reach the server, and flushes the
backlog once connectivity is restored.

**Auto-update**: the agent ships with `auto_update: true` by default. It periodically checks for a
new version, downloads it, and restarts the systemd service to apply it. This shows up in
`journalctl -u pulse-agent` as a `Stopping...` / `Started...` pair, sometimes with a version bump
(e.g. `v6.0.0-rc.5` → `v6.0.0-rc.6`). The restart causes a short burst of "flushing buffered
reports" log lines as the agent catches up — normal, and self-resolves within a minute or two.

## When to use it / trade-offs

- Good fit when you want a single dashboard across several self-hosted hosts (metrics + Docker
  container health + alerting) without standing up a heavier observability stack.
- The agent-pushes-to-server model means the server never needs inbound access to monitored
  hosts — only outbound HTTPS from each agent — which simplifies firewalling in a segmented network.
- The trade-off is that **the whole system is timestamp-dependent**: every report is stamped with
  the *agent host's* own clock, not the server's. Any clock drift on a monitored host directly
  corrupts that host's apparent freshness (see Pitfalls).

## Pitfalls

### "Host shows as stale/offline but the agent is clearly running"

This is almost always a **clock problem on the monitored host**, not a Pulse or network problem.
Every agent report is timestamped using the host's own system clock. If that clock has drifted
(commonly because `systemd-timesyncd` can't reach its configured NTP server and silently gives up),
the server keeps receiving reports — but with timestamps minutes in the past — so the host never
looks "current" and is shown as stale/offline indefinitely.

**Checklist before assuming an agent/network issue:**

1. On the affected host: `timedatectl status` → check `System clock synchronized: yes/no`.
2. `journalctl -u systemd-timesyncd` → look for repeated `Timed out waiting for reply from
   <ip>:123` messages.
3. Compare `date -u +%s` on the affected host vs. a known-good host (e.g. the Pulse server
   itself). A constant offset of more than a few seconds is the smoking gun.
4. Fix the underlying NTP issue, or temporarily point `systemd-timesyncd` at a public NTP pool.
   Once the clock resyncs, the host's "last seen" catches up to real time within a poll cycle or
   two.

Only once the clock is confirmed correct is it worth digging into `journalctl -u pulse-agent` for
actual send/connectivity errors (`401 Unauthorized`, `404 Not Found`, etc.).

### `404 Not Found` on the remote-config endpoint

You may see warnings like:

```
"Remote config request returned non-success status", "status_code":404, "message":"Failed to fetch remote config - using local (or previously cached) defaults"
```

This is generally harmless — the agent falls back to its local/cached configuration and continues
operating normally. It is **not** the cause of a host going stale.

### The local API requires authentication

Querying the Pulse server's REST API without a valid session/token returns
`{"error":"Authentication required"}`. This is expected behaviour, not a misconfiguration — the
API is intentionally locked down.

### Stale fields inside snapshot files

Pulse persists several JSON snapshot files (host/runtime caches, continuity tracking, AI
correlation data, etc.). Some contain `lastSeen`-style fields that are leftovers from an old format
and are **not** kept up to date — don't use them to judge whether a host is actually online. The
continuity-tracking data (updated live as agent reports arrive) is the authoritative source for
"last seen".

### Docker health alerts vs. host staleness

A container-level "Docker container health alert raised" for a given host can keep firing in real
time even while that same host's overall "last seen" looks stale — the docker-report path and the
host-report path are reported (and can lag) independently. Don't assume the whole agent is broken
just because one signal looks current and another doesn't.

## Related

<None yet — link the entity for the Pulse server/agents running on your own hosts here.>

## Sources

- [Legacy wiki.js: Pulse monitoring pitfalls](../../../../../sources/administration/services/2026-09-28-pulse-monitoring-pitfalls.md) — private source
