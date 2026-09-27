---
type: guide
title: Headless browser automation in a rootless container
description: 'Getting a CDP-driven headless Chrome/Chromium working inside a rootless container: missing shared libraries and a fontconfig crash.'
tags:
- browser-automation
- chrome
- docker
- rootless
- fontconfig
- cdp
status: draft
resource:
created: 2026-09-27T19:48:56Z
updated: 2026-09-27T19:48:56Z
generated:
  by: claude/sonnet-5
  at: 2026-09-27T19:48:56Z
verified: []
stale_after: 2027-09-27T19:48:56Z
sources:
- id: 2026-09-27-rootless-browser-automation-fontconfig
  resource: private:/sources/ai/tooling/2026-09-27-rootless-browser-automation-fontconfig.md
relations: []
superseded_by:
---

# Headless browser automation in a rootless container

A CLI/daemon that drives headless Chrome over the Chrome DevTools Protocol (CDP) — e.g. a
"browser use" style tool an AI agent shells out to — normally relies on the distro package
manager (`apt install --with-deps` or similar) to pull in Chrome's ~24 required shared libraries.
A **rootless** container (no `sudo`, no `apt`, no privileged package install) breaks that
assumption in two separate, easy-to-conflate ways: missing libraries, and a font-configuration
crash that only shows up once the libraries are present.

## Prerequisites

- A rootless container/user with outbound network access (to download a Debian package index and
  `.deb` files) but no root/`apt`.
- A CDP-capable headless Chrome/Chromium build already downloadable in user space (many browser
  automation CLIs offer a "download only, no system install" mode).

## Steps

### 1. Install the automation CLI and download the browser binary only

Use whatever "binary-only" install mode the automation tool offers — this step alone typically
needs no root, since it is a plain download. It is only the *subsequent* system-library
installation step that needs root/`apt` and will fail.

### 2. Resolve and unpack the missing shared libraries without root

`dpkg-deb -x` extracts a `.deb` archive's contents to an arbitrary directory and needs no
privileges — only *installing* a package via `dpkg -i`/`apt` does. So:

1. Find the browser build's own dependency manifest (many "Chrome for Testing"-style downloads
   ship one next to the binary) or derive the dependency closure from the target distro's package
   index.
2. Download the resulting `.deb` files.
3. `dpkg-deb -x <file>.deb <prefix>/` each one into a single user-writable prefix directory.

This reconstructs a working `/usr/lib`-equivalent tree without ever needing root.

### 3. Wrap the browser binary to inject `LD_LIBRARY_PATH` and `FONTCONFIG_FILE`

Automation CLIs usually exec a fixed binary name (e.g. `chrome`) inside a versioned directory.
Rename the real binary and replace it with a small wrapper script that points the dynamic linker
and fontconfig at the unpacked prefix before exec'ing the real binary:

```bash
CHDIR=<path to the versioned browser directory>
mv "$CHDIR/chrome" "$CHDIR/chrome-real"
cat > "$CHDIR/chrome" <<'EOF'
#!/bin/sh
DEPS="<unpacked prefix>/usr/lib/x86_64-linux-gnu"
export LD_LIBRARY_PATH="$DEPS:$LD_LIBRARY_PATH"
export FONTCONFIG_FILE="<unpacked prefix>/fontconfig/fonts.conf"
exec "$(dirname "$0")/chrome-real" --no-sandbox "$@"
EOF
chmod 755 "$CHDIR/chrome" "$CHDIR/chrome-real"
```

### 4. Generate a minimal fontconfig config

This is the step that is easy to miss, because the failure it prevents looks like a *connectivity*
problem rather than a *crash* (see Troubleshooting). A rootless install has no system
`/etc/fonts/`, so Chromium's font manager (Skia) has zero configured font directories and aborts
with `SIGABRT` instead of degrading gracefully. Point it at any font package that got pulled in
transitively while unpacking dependencies (e.g. `fonts-liberation`):

```bash
mkdir -p <unpacked prefix>/fontconfig <unpacked prefix>/fc-cache
cat > <unpacked prefix>/fontconfig/fonts.conf <<EOF
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "urn:fontconfig:fonts.dtd">
<fontconfig>
  <dir><unpacked prefix>/usr/share/fonts</dir>
  <cachedir><unpacked prefix>/fc-cache</cachedir>
</fontconfig>
EOF
```

### 5. Point the calling tool's own readiness check at the new binary, if it has one

Some higher-level tools have their own "is a browser installed" probe that only checks a
well-known default location (e.g. a Playwright cache directory) and won't know about a
custom-installed binary. If such a probe exists, point it at the new binary via whatever
environment variable it honors. This is often unrelated to whether the actual browser-driving
path works, but harmless and correct to set regardless.

## Verify

1. Run the automation CLI directly from a bare shell first (isolating it from any caller's own
   caching): e.g. `agent-browser open https://example.com` or equivalent.
2. Confirm the CDP listener stays up — the browser process should not exit within a second or two
   of printing its "DevTools listening on ws://..." line.
3. Only after step 1 succeeds standalone, retry through the higher-level tool that calls it.

## Troubleshooting

- **"Connect call failed" / CDP WebSocket handshake failure:** this is a *symptom*, not the root
  cause. It means the browser process exited (or never started) between opening its CDP port and
  the client connecting. Check whether the process is actually still alive after launch — a
  fontconfig `SIGABRT` looks exactly like "the port refused the connection" from the client side,
  because by the time the client connects, the process is already dead.
- **Diagnosing a post-launch crash:** temporarily redirect the real binary's stderr to a log file
  in the wrapper script, reproduce the failure, and look for `FATAL:` / `Received signal <N>`
  lines. A `Received signal 6` (SIGABRT) with a Skia/fontconfig frame in the message means step 4
  above is missing or misconfigured. Font-shaping warnings (e.g. harfbuzz "TextRunHarfBuzz" lines)
  are benign noise from missing font-shaping data and can be ignored.
- **A caller-side "CLI not found" error that persists after installing the CLI:** some callers
  cache a "not installed" result in-process (e.g. via `functools.lru_cache` around the discovery
  check) the first time they probe for the binary. Installing it afterward does not invalidate
  that cache — only a fresh process (restart the caller/container) re-probes.
- **IPv6-first `localhost` resolution is a real but separate pitfall.** If `/etc/hosts` lists an
  IPv6 loopback entry before the IPv4 one, and the browser's CDP port binds IPv4-only, any code
  path that resolves the literal string `localhost` (rather than using an explicit `127.0.0.1`
  URL) can fail intermittently. Worth ruling out with a direct socket test on both addresses, but
  don't assume it's the cause without confirming the URL the client actually connects to is not
  already a bare IP.

## Related

- Generalizes to any CDP-driven browser-automation tool (not just one specific CLI) running in any
  container runtime without root package-install privileges.

## Sources

- [Original setup notes (private)](../../../../../sources/ai/tooling/2026-09-27-rootless-browser-automation-fontconfig.md)
