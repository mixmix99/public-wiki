---
type: concept
title: Bogus MCE hardware-error events on AMD Zen 3 with Linux 7.0
description: How to recognise bogus 'System Fatal' machine-check log entries on AMD Zen 3 under Linux 6.19/7.0 and tell them from real hardware faults.
tags:
- linux
- amd
- zen3
- mce
- kernel
- proxmox
status: draft
resource:
created: 2026-11-08T15:18:53Z
updated: 2026-11-08T15:18:53Z
generated:
  by: claude/sonnet-5.5
  at: 2026-11-08T15:18:53Z
verified: []
stale_after: 2028-11-07T15:18:53Z
sources:
- id: proxmox-forum-ryzen-5950x-mce-kernel-7
  resource: https://forum.proxmox.com/threads/ryzen-5950x-uncorrected-software-containable-error-events-after-upgrade-to-kernel-7-0-x.184025/latest
- id: phoronix-linux-7-bogus-hardware-errors-zen3
  resource: https://www.phoronix.com/news/Linux-Hardware-Errors-AMD-Zen-3
- id: lkml-zen3-l3-scrubber-deferred-errors
  resource: https://lkml.iu.edu/hypermail/linux/kernel/2604.0/05457.html
- id: linux-edac-reset-per-poll-state
  resource: https://ratatoskr.run/linux-edac/2026/08/17478951/t
- id: 2026-11-08-amd-zen3-bogus-mce-linux-7
  resource: private:/sources/administration/proxmox/2026-11-08-amd-zen3-bogus-mce-linux-7.md
relations: []
superseded_by:
---

# Bogus MCE hardware-error events on AMD Zen 3 with Linux 7.0

On some AMD Zen 3 machines (reported on Ryzen 5000 desktop parts) running Linux 6.19 or 7.0, the
kernel log fills with `[Hardware Error]` blocks, sometimes labelled "System Fatal error", although
the machine keeps running normally. The data in these entries is invalid, which points to a
kernel reporting problem rather than failing hardware.

## How it works

Linux periodically polls the CPU's machine-check banks (counter `MCP` in `/proc/interrupts`) and
logs anything that looks like an error. A real uncorrectable error raises a machine-check
*exception* (counter `MCE`) and normally halts or resets the machine. Since the AMD MCA polling
and deferred-error handling were reworked around Linux 6.19, polling can pick up stale or
meaningless register contents on Zen 3 and log them as errors. Maintainers have traced related
cases to deferred-error status registers that do not match the documentation, and fixed some
with a Zen 3 filter. A later patch addresses state from one bank leaking into the next poll
(found by code inspection, not reproduced on hardware).

**Typical signature of a bogus entry:**

- `MCx_STATUS` is `0xffffffffffffffff`, or looks like a kernel address (`0xffff8...`).
- `Error Addr`, `IPID` and `Syndrome` are all `0x0`.
- The kernel says "Bank N is reserved", and N changes from one event to the next.
- Every flag is set at once (`Over|UE|MiscV|AddrV|PCC|SyndV|UECC|Deferred|Poison`).
- The system does not crash, freeze or reboot; the `MCE` counter in `/proc/interrupts` stays 0.
- The same signature shows up on other hosts after the move from kernel 6.17 to 7.0.

## When to use it / trade-offs

Use this to decide whether to worry, and what to do next:

| Situation | Reading | Action |
|---|---|---|
| Signature above, `MCE` counter 0, no crashes | Almost certainly bogus | Wait for a kernel fix; keep the history with `rasdaemon` |
| Entries stop on an older kernel (for example 6.17) | Confirms the kernel theory | Stay on the old kernel until a fix, if the noise matters |
| Non-zero `IPID` or `Error Addr`, a real bank name, errors on several CPUs, segfaults, host crashes | Possibly real | Test memory and CPU, update BIOS/microcode, consider replacing parts |

Workarounds and their cost:

- **Roll back to a 6.17 series kernel.** Reported to stop the uncorrected entries; corrected and
  deferred ones can still appear. Costs a reboot and means missing newer kernel fixes.
- **`ignore_ce` / `dont_log_ce`** (sysfs or kernel command line). Hides noise but also hides real
  corrected errors; not recommended as a permanent fix.
- **Do nothing and wait.** Reasonable when no other symptoms exist.

## Pitfalls

- **Do not start an RMA on these entries alone.** In one case a CPU was replaced and the same
  signature stayed; the entries named the same logical CPU each time, but the register values were
  physically impossible.
- The "always the same CPU" pattern is not explained in the sources; do not read it as proof of a
  weak core.
- A hardware-error line labelled "System Fatal" does not mean the system was fatally affected.
  Check the exception counter and uptime first.
- The journal rotates: install `rasdaemon` early if you want a complete history
  (`ras-mc-ctl --errors`).
- Older microcode and BIOS did not prevent or cure it for other reporters; updating them is
  housekeeping, not a known fix.

## Related

- [Lower idle power with the amd-pstate-epp balance_power profile](../../../guides/administration/linux/amd-pstate-epp-power-profile.md)
  (same CPU family and kernel, different topic)

## Sources

- [Proxmox forum: Ryzen 5950X events after upgrade to kernel 7.0.x](https://forum.proxmox.com/threads/ryzen-5950x-uncorrected-software-containable-error-events-after-upgrade-to-kernel-7-0-x.184025/latest)
- [Phoronix: Linux 7.0 last-minute fix for bogus hardware errors on AMD Zen 3](https://www.phoronix.com/news/Linux-Hardware-Errors-AMD-Zen-3)
- [LKML: L3 cache scrubber deferred errors flood logs on Zen 3 after 6.19 polling changes](https://lkml.iu.edu/hypermail/linux/kernel/2604.0/05457.html)
- [linux-edac: x86/mce/amd reset per-poll state patch](https://ratatoskr.run/linux-edac/2026/08/17478951/t)
- [Original notes (private)](../../../../../sources/administration/proxmox/2026-11-08-amd-zen3-bogus-mce-linux-7.md)
