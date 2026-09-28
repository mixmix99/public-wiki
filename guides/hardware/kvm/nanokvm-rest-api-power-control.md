---
type: guide
title: 'NanoKVM REST API: authentication and power control'
description: Authenticating against the Sipeed NanoKVM REST API (AES-encrypted login, JWT cookie) and scripting ATX power/reset via GPIO, including a generic JWT-secret-on-restart pitfall.
tags: [kvm, remote-control, hardware, api, jwt, upsnap, nanokvm]
status: draft
resource:
created: 2026-09-28T17:01:30Z
updated: 2026-09-28T17:01:30Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:01:30Z
verified: []
stale_after: 2027-09-28T17:01:30Z
sources:
- id: 2026-09-28-nanokvm
  resource: 'private:/sources/hardware/kvm/2026-09-28-nanokvm.md'
relations: []
superseded_by:
---

# NanoKVM REST API: authentication and power control

Script ATX-style power/reset control (and other actions) against a [NanoKVM](https://github.com/sipeed/NanoKVM)
over its REST API, without going through the web UI — useful for wiring a NanoKVM into
[UpSnap](https://github.com/seriousm4x/UpSnap) or similar wake/shutdown automation.

[NanoKVM](https://github.com/sipeed/NanoKVM) is a compact, open-source IP-KVM made by
[Sipeed](https://wiki.sipeed.com/). It provides remote keyboard/mouse/video (HDMI capture) access to
a machine independent of its OS, plus ATX-style power/reset button control via GPIO wires connected
to the host's front-panel header — the same use case as [PiKVM](pikvm-cli-and-maintenance.md), in a
smaller, cheaper (RISC-V based) form factor.

- **Product page:** [sipeed.com — NanoKVM](https://www.sipeed.com/product/nanokvm)
- **Official wiki:** [wiki.sipeed.com/hardware/en/kvm/NanoKVM](https://wiki.sipeed.com/hardware/en/kvm/NanoKVM/introduction.html)
- **Source code:** [github.com/sipeed/NanoKVM](https://github.com/sipeed/NanoKVM)

## Prerequisites

- A NanoKVM reachable on the network, wired to the target host's ATX front-panel header.
- `curl`, `openssl`, and (for URL-encoding) `node` or an equivalent — or the Python client mentioned
  below.
- Default credentials, if never changed: web UI `admin`/`admin`, SSH `root`/`root` (documented
  vendor defaults — change both after initial setup). **Changing the web UI password does not
  automatically change the SSH password** — they are independent, unlike PiKVM.

## How authentication works

Unlike PiKVM (which accepts plain HTTP Basic Auth on every request), NanoKVM requires a login step
that returns a JWT, which must then be sent back as a cookie on every subsequent request. There is
no Basic Auth fallback and no API-key mode.

Base URL: `http://<nanokvm-ip>/api/`

## Steps

### 1. Encrypt the password

The frontend encrypts the password client-side before sending it to `/api/auth/login`, using
AES-256-CBC with a hardcoded passphrase (`nanokvm-sipeed-2024`), OpenSSL-compatible key derivation
(`EVP_BytesToKey`, MD5, salted), then Base64-encodes and URL-encodes the result:

```bash
ENC=$(echo -n '<password>' | openssl enc -aes-256-cbc -a -salt -md md5 -pass pass:nanokvm-sipeed-2024)
ENC_URLENC=$(node -e "console.log(encodeURIComponent(process.argv[1]))" "$ENC")
```

This is **not** meaningful security — the key is public in the NanoKVM source
(`web/src/lib/encrypt.ts`) — it just obfuscates the password from being sent in the clear over plain
HTTP. Sending the plain password (or an MD5 hash of it) is rejected; it must be encrypted exactly
this way.

### 2. Log in

```bash
curl -s -X POST http://<nanokvm-ip>/api/auth/login \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"admin\",\"password\":\"$ENC_URLENC\"}"
# → {"code":0,"msg":"success","data":{"token":"<jwt>"}}
```

### 3. Use the token

Every other endpoint requires the JWT sent back as a `nano-kvm-token` **cookie**:

```bash
curl -s -X POST http://<nanokvm-ip>/api/vm/gpio \
  -H "Cookie: nano-kvm-token=<jwt>" \
  -H "Content-Type: application/json" \
  -d '{"type":"power","duration":800}'
```

Some documentation (e.g. third-party API reference projects) claims the token can also be passed via
an `Authorization: Bearer <jwt>` header — in practice, against current NanoKVM firmware, only the
cookie works. Test against your own device's firmware before relying on the header.

### 4. Extend token lifetime for unattended automation (optional)

The JWT's lifetime is controlled by `jwt.refreshTokenDuration` (seconds) in `/etc/kvm/server.yaml`
**on the NanoKVM device itself**, default `2678400` (31 days). To avoid re-authenticating on every
scripted call, extend it (e.g. to 10 years) and generate a token once, reused indefinitely:

```bash
# On the NanoKVM device, as root:
sed -i 's/refreshTokenDuration: 2678400/refreshTokenDuration: 315360000/' /etc/kvm/server.yaml
/etc/init.d/S95nanokvm restart
```

Then repeat the login step above — the returned token carries the new expiry. This changes the
lifetime for *all* future logins, including the web UI session.

### 5. GPIO / power control

```
POST /api/vm/gpio
```

| Field | Type | Notes |
|---|---|---|
| `type` | string | `"power"` or `"reset"` — which ATX header wire to pulse |
| `duration` | integer | Milliseconds to hold the pulse. `800` for a short/normal press, `5000`+ for a forced power-off (most BIOSes trigger hard power-off after ~4s held) |

```bash
curl -s -X POST http://<nanokvm-ip>/api/vm/gpio \
  -H "Cookie: nano-kvm-token=<jwt>" \
  -H "Content-Type: application/json" \
  -d '{"type":"power","duration":800}'
# → {"code":0,"msg":"success","data":null}
```

> [!warning] Dumb button press, not a state-aware power action
> Unlike PiKVM's ATX API (`POST /api/atx/power?action=on`, which checks current power state and
> no-ops if the target is already in the requested state), NanoKVM's `gpio` endpoint just simulates a
> physical button press regardless of current state. Pressing it on a machine that's already on will
> typically trigger an ACPI graceful-shutdown request, not do nothing. Check the target's actual
> power state (e.g. via ping, or the video capture) before firing this — don't assume idempotency.
> This matters a lot for wake/shutdown automation (e.g. UpSnap), where an unguarded "wake" call on an
> already-online host can shut it down instead.

## Other useful endpoints

The web UI (browser dev tools → Network tab) is the most reliable way to discover further endpoints,
since NanoKVM has no published OpenAPI spec. A few commonly referenced ones (all require the same
cookie auth as above):

| Endpoint | Purpose |
|---|---|
| `GET /api/vm/info` | Device info |
| `GET /api/vm/hardware` | Hardware/capability info |
| `GET /api/vm/gpio` | Current GPIO/power-LED state |
| `GET /api/vm/image` | List mountable ISO images |
| `GET /api/auth/account` | Current logged-in account info |

## Client libraries

For scripting beyond simple curl one-liners, consider
[`python-nanokvm`](https://github.com/puddly/python-nanokvm) (`pip install nanokvm`) — an async
Python client that handles the login/encryption/token dance internally:

```python
from nanokvm.client import NanoKVMClient
from nanokvm.models import GpioType

async with NanoKVMClient("http://<nanokvm-ip>/api/") as client:
    await client.authenticate("admin", "<password>")
    await client.push_button(GpioType.POWER, duration_ms=800)
```

## Verify

```bash
curl -s http://<nanokvm-ip>/api/vm/info -H "Cookie: nano-kvm-token=<jwt>"
# → 200 with device info if the token is valid
```

## Troubleshooting

### Known issue — non-persistent JWT secret invalidates tokens on restart

`/etc/kvm/server.yaml`'s `jwt.secretKey` ships **empty**:

```yaml
jwt:
    secretKey: ""
    refreshTokenDuration: 315360000
    revokeTokensOnLogout: true
```

When empty, `NanoKVM-Server` generates a **random** signing key at process startup. Every JWT issued
before that startup — no matter how long its own `exp` claim says it's valid for — stops verifying
and gets `401 unauthorized`, even though the token itself looks unexpired.

This matters because `NanoKVM-Server` can restart on its own outside of any manual action or
firmware update — e.g. correlated with an HDMI capture chip failing to detect a signal (plausible
whenever the target host has no video output, such as being powered off). For anything relying on a
"long-lived" NanoKVM token (unattended wake/shutdown automation being the main case), this means the
token can go stale silently, and tends to do so exactly when the target host is off and the token is
needed to wake it.

**Fix:** set a fixed `secretKey` so tokens survive restarts:

```sh
# On the NanoKVM device, as root:
sed -i 's/secretKey: ""/secretKey: "<a-random-64-char-hex-string>"/' /etc/kvm/server.yaml
/etc/init.d/S95nanokvm restart
```

Generate the random value with `openssl rand -hex 32`. After the restart, log in again once to issue
a token signed with the new persistent key — every token issued from then on survives future
restarts.

Even with `secretKey` fixed, the underlying restart trigger (if firmware-side, e.g. an HDMI-probe
crash loop) is unresolved by this fix — it just makes those restarts non-fatal for auth. If a stored
automation token stops working again after applying this fix, suspect a firmware update having reset
`/etc/kvm/server.yaml` back to an empty `secretKey`.

A belt-and-suspenders mitigation for unattended automation is a small script that re-logs in
periodically and pushes the fresh token to whatever consumes it (e.g. UpSnap), rather than relying on
one static value indefinitely — closes the gap if `secretKey` ever reverts unnoticed.

## Related

- [PiKVM: CLI maintenance reference](pikvm-cli-and-maintenance.md) — the other common IP-KVM
  platform, with a state-aware ATX power API and Basic Auth instead.

## Sources

- [Legacy wiki.js: NanoKVM REST API and UpSnap integration](../../../../../sources/hardware/kvm/2026-09-28-nanokvm.md) — private source (original capture)
- [NanoKVM GitHub repository](https://github.com/sipeed/NanoKVM)
- [NanoKVM official wiki](https://wiki.sipeed.com/hardware/en/kvm/NanoKVM/introduction.html)
- [python-nanokvm client library](https://github.com/puddly/python-nanokvm)
