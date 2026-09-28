---
type: concept
title: PCI address drift breaking hypervisor passthrough
description: Why PCIe device addresses can shift between reboots and silently break GPU/NIC passthrough configs that pin a fixed bus address, and how to detect and mitigate it.
tags:
- proxmox
- pci-passthrough
- vfio
- kvm
- hypervisor
status: draft
resource:
created: 2026-09-27T19:54:26Z
updated: 2026-09-28T17:06:08Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:06:08Z
verified: []
stale_after: 2028-09-26T19:54:26Z
sources:
- id: pci-address-drift-passthrough
  resource: private:/sources/administration/proxmox/2026-09-27-pci-address-drift-passthrough.md
relations: []
superseded_by:
---

# PCI address drift breaking hypervisor passthrough

PCIe bus/device numbers (e.g. `0a:00.0`) are assigned **dynamically by BIOS/UEFI during enumeration**, not fixed by physical slot. When a hypervisor's VM configuration pins a device for passthrough by its PCI address (e.g. `hostpci0: 0000:0a:00` in Proxmox), a change in enumeration order between boots can hand a completely different device to the VM than intended.

## How it works

During POST, firmware walks the PCIe tree (typically depth-first) and assigns bus numbers as it discovers devices. The order devices become "ready" to answer enumeration can shift between boots for reasons that have nothing to do with physical topology — firmware settings that change device init timing, link training variance, or add-in-card firmware differences. Two unrelated devices (e.g. a GPU and a 10G NIC) can end up swapping bus addresses across a reboot even though neither was physically moved.

If a VM's passthrough config hard-codes the address one device happened to have, and that address is later claimed by a different device, the hypervisor's `vfio-pci` driver binds to whatever device now sits there — silently detaching it from the host (or from the driver the host expects to use it), and handing it to the VM instead. If the device that got silently reassigned is something the host itself depends on (a production NIC, a storage controller), this can look like a sudden, unexplained loss of connectivity or storage the moment the VM using that passthrough config starts.

## When to use it / trade-offs

This risk exists any time a passthrough config references a device by **raw PCI bus address** rather than a more stable identifier. Mitigations, roughly in order of robustness:

- **Pin devices by a stable identifier where the hypervisor supports it** — vendor/device ID plus PCI domain, an IOMMU group reference, or (best) a `/dev/disk/by-id`-style persistent path for storage. Not all hypervisors or device classes support this for every passthrough scenario.
- **Disable "fast boot" / "skip POST check" style BIOS settings** that reduce device re-enumeration time — these are a plausible trigger for enumeration-order changes, though not proven causal in every case; treat correlation (setting enabled the same day drift appeared) as a strong lead, not certainty, and verify PCI addresses across several boots with the setting on before concluding it's safe.
- **Spot-check PCI addresses after every reboot** on hardware known to be affected (`lspci -nn` compared against a documented baseline) until stability is proven across multiple consecutive boots.
- **Avoid passing through devices the host itself needs to function** (a production NIC, boot storage) if at all avoidable — the failure mode is much less severe when the swapped-in device is something the host doesn't depend on.

## Pitfalls

- **A single stable reboot is not proof of a fix.** Enumeration-order drift is inherently boot-to-boot timing variance; treat any "it's fixed now" conclusion as provisional until confirmed over several boots.
- **The device that silently loses its driver binding may not fail loudly.** A NIC losing its driver to `vfio-pci` can look like "the network died" with no obvious PCI-level error message pointing at the actual cause — check `dmesg` for the driver unbind/rebind sequence, not just the symptom.
- **VM startup order matters.** If the VM with the passthrough config auto-starts early in the boot sequence, the window between "host driver claims the device" and "VM startup script re-claims it for vfio-pci" can be very short, making the failure look intermittent or timing-dependent.

## Related

- Concept: [NVIDIA open kernel module dropping Pascal GPU support](../truenas/nvidia-open-driver-pascal-support.md) — a separate, driver-level GPU passthrough failure mode sometimes found in the same investigation session as PCI address drift.
- Guide: [GPU passthrough on Proxmox VE](../../../guides/administration/proxmox/gpu-passthrough-setup.md) — the passthrough setup this failure mode most commonly breaks.

## Sources

- [Legacy wiki.js (private): boot network failure and PCI address drift on a Proxmox host](../../../../../sources/administration/proxmox/2026-09-27-pci-address-drift-passthrough.md) — private source (real incident)
