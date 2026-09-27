---
type: guide
title: On-demand wake and idle shutdown for a reverse-proxied host via Traefik
description: Use a Traefik middleware plugin plus a wake/shutdown API to wake a sleeping backend on first request and power it back down after inactivity.
tags: [traefik, wake-on-lan, wol, upsnap, power-management, reverse-proxy]
status: draft
resource:
created: 2026-09-27T19:49:16Z
updated: 2026-09-27T19:49:16Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T19:49:16Z
verified: []
stale_after: 2027-09-27T19:49:16Z
sources:
- id: 2026-09-27-traefik-wol-setup
  resource: 'private:/sources/administration/services/2026-09-27-traefik-wol-setup.md'
relations: []
superseded_by:
---

# On-demand wake and idle shutdown for a reverse-proxied host via Traefik

Wake a sleeping or powered-off backend automatically on its first proxied request (instead of
returning a connection error), and shut it back down after a period of inactivity — hands-off
power management for a LAN service that doesn't need to run 24/7, such as a GPU inference host.

## Prerequisites

- Traefik as reverse proxy, with the [experimental plugin
  system](https://doc.traefik.io/traefik/plugins/) enabled.
- A wake/power-management API that can wake and shut down the target host — e.g.
  [UpSnap](https://github.com/seriousm4x/UpSnap) (Wake-on-LAN, or an IP-KVM's ATX API for hosts
  where WOL is unreliable), or any similar service with an HTTP wake/shutdown endpoint.
- The [traefik-wol](https://github.com/MarkusJx/traefik-wol) plugin (or an equivalent Traefik
  middleware plugin that supports a health-check-then-wake pattern).
- A lightweight, always-fast HTTP health-check endpoint on the target host — **not** the actual
  service's own port if that service's response time can spike under load (see Pitfalls).

## Steps

### 1. Register the plugin and add a loopback-only internal entryPoint

Static Traefik config (requires a full restart to apply — this section is not hot-reloaded):

```yaml
entryPoints:
  web:
    # ... existing HTTP entrypoint, force-redirects to HTTPS ...
  websecure:
    # ... existing HTTPS entrypoint, needs a valid cert ...
  internal:
    address: "127.0.0.1:<port>"

experimental:
  plugins:
    wol:
      moduleName: github.com/MarkusJx/traefik-wol
      version: <version>
```

A dedicated loopback-only `internal` entryPoint matters because most wake/shutdown middleware
plugins call plain `http.Get()`/`http.Post()` with no TLS or redirect handling: your public HTTP
entrypoint likely force-redirects to HTTPS (the plugin would just hit the redirect), and your
HTTPS entrypoint needs a certificate the plugin's bare HTTP client usually can't validate for a
loopback call. A plain-HTTP, no-redirect, `127.0.0.1`-bound entryPoint sidesteps both problems —
and since it's loopback-only, it's not reachable from outside the Traefik container/host, so it
needs no separate access control.

### 2. If your wake API requires auth the plugin can't send

Many wake-on-demand middleware plugins offer no way to set custom HTTP headers on their
start/stop requests. If your wake API requires a bearer token or similar, don't point the plugin
at it directly — point it at an **internal Traefik route** instead, which injects the header and
rewrites the path to the real API call:

```yaml
http:
  routers:
    wake-proxy-router:
      rule: "PathPrefix(`/wake/<host>`)"
      entryPoints: [internal]
      service: wake-proxy-service
      middlewares: [wake-proxy-path, wake-proxy-auth]

  middlewares:
    wake-proxy-auth:
      headers:
        customRequestHeaders:
          Authorization: "Bearer <token>"
    wake-proxy-path:
      replacePath:
        path: "/api/wake/<device-id>"

  services:
    wake-proxy-service:
      loadBalancer:
        servers:
          - url: "http://<wake-api-host>:<port>"
```

Traefik authenticates to the wake API on the plugin's behalf; the plugin itself only ever talks to
`127.0.0.1`. Duplicate this pattern for a shutdown route if your wake API also exposes shutdown.

### 3. Add the WOL middleware to the actual proxied route

```yaml
http:
  routers:
    my-service-router:
      rule: "Host(`<my-service-host>`)"
      entryPoints: [web, websecure]      # explicit — omitting this defaults to ALL entryPoints, including `internal`
      service: my-service
      middlewares: ["my-service-wol"]
      tls:
        certResolver: <resolver>

  middlewares:
    my-service-wol:
      plugin:
        wol:
          healthCheck: "http://<target-ip>:<health-port>"
          startUrl: "http://127.0.0.1:<internal-port>/wake/<host>"
          startMethod: "GET"
          numRetries: 12
          requestTimeout: 15
          stopUrl: "http://127.0.0.1:<internal-port>/shutdown/<host>"
          stopMethod: "GET"
          stopTimeout: 60      # check your plugin's docs for the unit — see Troubleshooting

  services:
    my-service:
      loadBalancer:
        servers:
          - url: "http://<target-ip>:<target-port>"
```

On each request: the health check is called first; if it responds, the request proxies straight
through with negligible added latency; if not, `startUrl` is called and the health check retried
up to `numRetries` times before proxying through. If `stopUrl` is set, an idle timer resets on
every proxied request and fires the shutdown call once it's been inactive for `stopTimeout`,
provided the health check still shows the host up.

## Verify

```bash
curl -sk https://<my-service-host>/<some-endpoint>     # first request wakes the host, waits for it to answer
curl -sk https://<my-service-host>/<some-endpoint>     # subsequent requests pass through immediately
```

Check the health-check-only field manually if you want to confirm the wake trigger without going
through the full router: from inside the Traefik container/host, hit the internal wake route
directly — but see the warning below first.

## Troubleshooting

- **Check whether your wake/shutdown API's action endpoints are idempotent before assuming it's
  safe to poll or "just check" one.** Some wake APIs unconditionally execute the wake/shutdown
  action on every call, with no guard against the host already being in that state — calling
  "wake" on an already-running host can trigger a real physical power-button action instead of a
  no-op. If your API has both a single-device action endpoint and a bulk/group endpoint, check
  whether only one of them filters by current status — that mismatch is easy to miss and easy to
  hit by accident during setup.
- **A timer/duration-shaped config field may use a non-obvious unit.** Read your specific plugin's
  source or changelog for fields like an idle-shutdown timeout — some accept a duration string,
  some accept seconds, some accept a plain integer number of minutes. Getting this wrong either
  shuts the host down almost immediately or effectively never.
- **Don't point the health check at the proxied service's own port if that service's response time
  can spike under load.** A wake-on-demand middleware typically re-runs its health check on every
  incoming request with a fixed timeout. If the backend service is legitimately slow to respond
  under its own load (long-running requests, an internal work queue), the health check can time
  out even though the host is up and working — a false "it's asleep" reading. If that false
  negative reaches an unguarded wake endpoint (see the point above), the result is a spurious wake
  action fired at an already-running, busy host. Use a separate, trivial, always-fast health-check
  listener on the target host instead of its real service port, and give the timeout headroom
  beyond the service's worst normal-case latency.
- **`entryPoints` defaults to "all" if left unspecified** on a router — explicitly restrict a
  public-facing router to your public entrypoints so it doesn't also (harmlessly, but sloppily)
  pick up an internal/loopback entrypoint meant only for internal wake/shutdown routes.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: Traefik Wake-on-LAN setup](../../../../../sources/administration/services/2026-09-27-traefik-wol-setup.md) — private source (real incident and worked example)
- [traefik-wol plugin](https://github.com/MarkusJx/traefik-wol)
- [UpSnap](https://github.com/seriousm4x/UpSnap)
