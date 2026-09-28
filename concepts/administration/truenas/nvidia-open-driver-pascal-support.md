---
type: concept
title: NVIDIA open kernel module dropping Pascal GPU support
description: TrueNAS Scale (and other distros) moving to NVIDIA's open-source kernel module drops hardware support for Pascal-generation GPUs, silently breaking transcoding/CUDA workloads.
tags:
- nvidia
- gpu
- truenas
- linux
- drivers
- jellyfin
- transcoding
status: draft
resource:
created: 2026-09-27T19:54:27Z
updated: 2026-09-28T18:55:38Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:55:38Z
verified: []
stale_after: 2028-09-26T19:54:27Z
sources:
- id: truenas-pascal-gpu-driver
  resource: private:/sources/administration/truenas/2026-09-27-truenas-pascal-gpu-driver.md
- id: truenas-forums-nvidia-kernel-module-change
  resource: https://forums.truenas.com/t/nvidia-kernel-module-change-in-truenas-25-10-what-this-means-for-you/51070
- id: truenas-forums-pascal-gpu-25-10-3-1
  resource: https://forums.truenas.com/t/pascal-gpu-on-version-25-10-3-1/66392
relations: []
superseded_by:
---

# NVIDIA open kernel module dropping Pascal GPU support

NVIDIA ships two Linux kernel module variants: the long-standing **proprietary** driver, and a newer **open-source** kernel module (`nvidia.ko`, open variant) that NVIDIA has been pushing distributions and appliance vendors toward as the default. The open variant does not support the same range of GPU generations as the proprietary one — most notably, it drops **Pascal**-generation cards entirely.

## How it works

The open kernel module relies on GPU System Processor (GSP) firmware blobs bundled with the driver to talk to the hardware. As of the versions in question, NVIDIA only ships GSP firmware for **Turing and later** architectures (RTX 16xx/20xx and newer, Ampere, Ada, Blackwell) — there is no Pascal GSP firmware blob at all. A Pascal card (`GTX 10xx`, `Quadro Pxxxx`, e.g. a Quadro P4000/GP104) physically enumerates fine on the PCI bus, but the driver refuses to bind to it, logging something like:

```
NVRM: The NVIDIA GPU 0000:01:00.0 (PCI ID: 10de:1bb1)
NVRM: installed in this system is not supported by open
NVRM: nvidia.ko because it does not include the required GPU
```

`nvidia-smi` then fails with "couldn't communicate with the NVIDIA driver", and any application-level GPU picker (a NAS's app framework, a container orchestrator) simply doesn't list the card as available — there is often **no loud error surfaced to the end user**, just hardware transcoding or CUDA workloads silently falling back to CPU.

An appliance OS (like TrueNAS Scale) switching its bundled driver from proprietary to the open variant in a routine OS upgrade — especially one applied via an **automatic update mechanism** — can break GPU-dependent workloads with no configuration change on the user's part and no clear error message pointing at the cause.

## When to use it / trade-offs

- **The open kernel module is the vendor-recommended direction** for newer GPUs and is required for certain features (e.g. some confidential-computing / virtualization scenarios). It is not itself a mistake to move to.
- **The trade-off is generational**: any Pascal-or-older card is fully cut off, with no partial/degraded support — not slower, not CPU-fallback-with-warning, just entirely unrecognized by the open driver.
- **Auto-updating appliance OSes are a particular risk** for this failure mode, since the driver-generation switch can ride along inside what looks like a routine point release, with no explicit "your GPU will stop working" warning at update time.

## Pitfalls

- **Check the actual bound driver after any OS upgrade on a system with GPU passthrough or an NVIDIA card**, don't just trust that "the app still shows RUNNING" — a transcoding/CUDA app can keep running happily on CPU with zero indication in its own UI that hardware acceleration silently stopped working.
- **A spare "downgrade" GPU of the same or an adjacent generation may not help.** Multiple Pascal-generation cards (e.g. a GTX/Quadro from the GP10x family) all hit the identical wall — the cutoff is architectural (pre-Turing), not per-model.
- **There is currently no supported in-place workaround** other than staying on an older OS release that still ships the proprietary driver, or replacing the GPU with a Turing-or-later card. Reverting an appliance OS that auto-updates is often undesirable for other reasons, which can make "just don't use hardware acceleration anymore" the pragmatic outcome if the workload doesn't strictly need it (e.g. clients that already direct-play/direct-stream media rather than requiring transcoding).

## Related

- Concept: [PCI address drift breaking hypervisor passthrough](../proxmox/pci-address-drift-passthrough.md) — a separate, hypervisor-level GPU passthrough failure mode that can surface in the same investigation session as this driver-level one.
- Guide: [Passing a GPU through to a TrueNAS Scale app](../../../guides/administration/truenas/gpu-passthrough-app-selection.md) — the app-level GPU selection step that sits one layer above this driver-level failure mode; a GPU cut off by this issue never shows up as selectable there.

## Sources

- [Legacy wiki.js (private): Jellyfin GPU transcoding broken after TrueNAS 25.10 dropped Pascal support](../../../../../sources/administration/truenas/2026-09-27-truenas-pascal-gpu-driver.md) — private source (real incident)
- [NVIDIA Kernel Module Change in TrueNAS 25.10 — TrueNAS Forums](https://forums.truenas.com/t/nvidia-kernel-module-change-in-truenas-25-10-what-this-means-for-you/51070)
- [Pascal GPU on Version 25.10.3.1 — TrueNAS Forums](https://forums.truenas.com/t/pascal-gpu-on-version-25-10-3-1/66392)
