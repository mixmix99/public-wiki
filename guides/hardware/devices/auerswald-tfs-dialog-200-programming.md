---
type: guide
title: Programming the Auerswald TFS-Dialog 200 door intercom
description: Enter programming mode on an Auerswald TFS-Dialog 200 via a PBX phone and set call targets, door-opener, light, timing, volume and PIN.
tags: [auerswald, doorbell, intercom, pbx, dtmf]
status: draft
resource:
created: 2026-09-27T19:27:15Z
updated: 2026-09-27T19:27:15Z
generated:
  by: claude/opus-5.5
  at: 2026-09-27T19:27:15Z
verified: []
stale_after: 2027-09-27T19:27:15Z
sources:
- id: 2026-09-27-tfs-dialog-200-programming
  resource: private:/sources/hardware/devices/2026-09-27-tfs-dialog-200-programming.md
relations: []
superseded_by:
---

# Programming the Auerswald TFS-Dialog 200 door intercom

The Auerswald TFS-Dialog 200 is an analog door intercom (up to 4 bell buttons) that connects to an
a/b (analog) port of a PBX. It has no web UI: everything is programmed with DTMF digits from an
internal phone. This guide covers entering programming mode and every programming code from the
vendor manual (V07 02/2018).

## Prerequisites

- The TFS-Dialog 200 is connected to an internal analog port of the PBX and set up there with an
  internal number `<extension>` (see [PBX setup](#pbx-setup)).
- An internal analog phone with DTMF (tone dialling), or an ISDN phone with DTMF signalling.
- The device PIN — `0000` from the factory.

## Steps

### 1. Enter programming mode

1. Press any bell button (short tone) and **hold it for about 5 seconds** until a second tone sounds.
2. **Within 3 minutes**, pick up an internal phone and dial `<extension>`. The intercom answers.

### 2. Authenticate and enter a code

Every function is entered as:

```
*   <PIN>   *   <code digits>
```

1. Dial `*` — you hear a short tone.
2. Dial the PIN, then `*`. Wait for the **confirmation tone** (five rapid beeps).
3. Dial the function code (table below). Wait for the confirmation tone again, or hang up if the
   function says so.

Rules:

- If you dial `*` and immediately hear the five-beep confirmation, the device is **still in
  programming mode** — skip `<PIN>*` and go straight to the code.
- Several functions can be programmed in a row without hanging up; the PIN is remembered until
  programming mode ends.
- Programming mode ends after more than 3 minutes without input, or when a bell button is pressed.
- A wrong input gives a 1–2 s **busy tone** instead of the confirmation. Start again with `*`.

### 3. Function codes

`B` = bell button 1–4, numbered top to bottom. Times given as `1–9 × 0.5 s` mean: dial one digit,
the value is digit × 0.5 s.

**Bell buttons**

| Code | Function | Values | Default |
|---|---|---|---|
| `2 B <number>` then **hang up** | Call target of button B | up to 32 characters `0-9 * #` | 31, 32, 33, 34 |
| `3 B F` | Switching-module frequency triggered by button B | `0` none, `1`–`4` | 1, 2, 3, 4 |
| `4 B Z` | Extra bell (terminals 1/2) triggered by button B | `0` none, `1`, `2` | 1, 2, 0, 0 |
| `7 B L` | Button B also switches the stair light | `0` off, `1` on | off |

Notes on `2` (call target):

- Some PBXes need a dialling pause inside the number: stop for at least 5 s during input, a short
  tone confirms the stored pause.
- For no call target (e.g. to use the PBX's baby/senior call feature), hang up right after the
  button digit, or wait 5 s to store only a pause.
- After this function you must call the intercom again for further programming.

**Door opener and light**

| Code | Function | Values | Default |
|---|---|---|---|
| `5 6 T` | Door-opener on-time | `1`–`9` × 0.5 s | 2 s |
| `2 6 <digits> #` | Digits dialled after `#` during a door call to open the door | 1–6 digits | `#9` |
| `5 5 T` | Light on-time | `1`–`9` × 0.5 s | 0.5 s |
| `2 5 <digits> #` | Digits dialled after `#` during a door call to switch the light | 1–6 digits | `#8` |

The door-opener and light sequences must differ.

**Timing**

| Code | Function | Values | Default |
|---|---|---|---|
| `5 1 M` | Maximum talk time | `1`–`9` min, `0` unlimited | 3 min |
| `5 2 R` | Maximum ring duration | `1`–`9` × 10 s | 20 s |
| `5 3 T` | Ring delay (so repeated presses don't restart dialling) | `1`–`9` × 0.5 s | 0.5 s |
| `5 4 T` | Pause between hang-up and re-dial (PBX hook/flash time), when powered from the a/b port | `1`–`9` × 0.5 s | 2 s |

**Audio**

| Code | Function | Values | Default |
|---|---|---|---|
| `5 0 S` | a/b line input sensitivity | `0` low – `9` high | 3 |
| `5 8 A` | Ambient noise near the phones | `0` quiet, `1` loud | quiet |
| `5 7 V` | Intercom speaker volume | `0`–`9` | 2 |
| `5 9 S` | Beep on bell-button press | `0` off, `1` on | on |

If the beep is off, you only hear **one** tone when entering programming mode.

**PIN and reset**

| Code | Function | Notes |
|---|---|---|
| `2 9 <new PIN> # <new PIN> #` | Change the PIN | 1–6 digits, entered twice. Strongly recommended. |
| `9 1` then **hang up** | Factory reset | The PIN is **not** reset. Re-enter programming mode afterwards. |

### Example: ring extension `<target>` on bell button 1

```
hold bell button 1 for 5 s  →  dial <extension> on an internal phone
*          (1 short tone; if you hear 5 beeps, skip the next line)
0000 *     (factory PIN, wait for 5 beeps)
2 1 <target>
hang up
```

## Verify

Press the bell button. The phone(s) at the call target should ring within the ring delay. Answer
and dial `#9` (or your custom sequence) to check the door opener.

## Troubleshooting

- **Call is not ended after hanging up / speech path switches poorly:** the intercom does not
  detect the PBX busy tone. Adjust the a/b input sensitivity (`5 0`) first. Only change the
  ambient and speaker volume settings if that does not help. Also make sure the PBX sends a busy
  tone when a call ends on that port.
- **Busy tone right after a code:** the input was wrong. Start again with `*`.
- **Five beeps immediately after `*`:** still in programming mode. Leave out `<PIN>*`.

## PBX setup

The PBX must know the intercom as an internal subscriber before any of this works.

- **Auerswald COMpact 4000, COMpact 5000/R, COMmander 6000/R/RX (firmware ≥ 6.4A):** create it as
  a *door station* from the device template. The PBX then controls call targets and ring duration
  completely. Leave the button call targets in the intercom at their factory defaults (31–34).
- **Other PBXes:** create it as a normal analog extension and enable busy tone at the end of a call
  for that port.

Up to six optional a/b switching modules can be added to drive extra bells, a door opener or a
stair light (see `3 B F`).

## Sources

- [Legacy wiki.js: Auerswald TFS-Dialog 200 programming](../../../../../sources/hardware/devices/2026-09-27-tfs-dialog-200-programming.md) — private source, includes the vendor manual (German PDF, V07 02/2018)
