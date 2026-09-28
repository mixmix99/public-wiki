---
type: guide
title: GPU passthrough on Proxmox VE
description: 'Step-by-step GPU passthrough setup on Proxmox VE for AMD, Intel and NVIDIA GPUs: IOMMU, VFIO binding, VM assignment, and the Error 43 workaround.'
tags: [proxmox, gpu, passthrough, vfio, iommu, virtualization]
status: draft
resource:
created: 2026-09-28T17:03:37Z
updated: 2026-09-28T17:03:37Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:03:37Z
verified: []
stale_after: 2027-09-28T17:03:37Z
sources:
- id: 2026-09-28-gpu-passthrough-setup
  resource: 'private:/sources/administration/proxmox/2026-09-28-gpu-passthrough-setup.md'
relations: []
superseded_by:
---

# GPU passthrough on Proxmox VE

GPU passthrough gives a VM direct, exclusive access to a physical GPU, using VFIO to hand the PCI
device to the guest instead of the host's own driver. This is the standard way to get
near-native GPU performance in a VM — gaming, AI/ML workloads, video editing, or hardware-accelerated
transcoding — on Proxmox VE, for AMD, Intel or NVIDIA GPUs.

## Requirements and hardware compatibility

- **Proxmox VE 8.x or higher.**
- **CPU virtualization extensions with IOMMU support**: Intel VT-d, or AMD-Vi (AMD-V).
- **Motherboard/BIOS**: must support IOMMU; ideally also "Above 4G Decoding" and "Resizable BAR"
  (optional, performance-only).
- **GPU**:
  - AMD GPUs generally pass through the most cleanly.
  - NVIDIA consumer GPUs need a workaround for "Error 43" (see below); NVIDIA enterprise/datacenter
    GPUs may support vGPU/mediated-device passthrough instead of exclusive passthrough.
  - Intel integrated GPUs can be passed through too, with more limited use cases.

BIOS: enable IOMMU (VT-d/AMD-Vi), disable CSM for UEFI boot, optionally enable Above 4G
Decoding/Resizable BAR, and update to the latest BIOS version before starting.

## Steps

### 1. Enable IOMMU on the host

Edit `/etc/default/grub` and set, depending on CPU vendor:

```bash
# Intel
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt"
# AMD
GRUB_CMDLINE_LINUX_DEFAULT="quiet amd_iommu=on iommu=pt"
```

```bash
update-grub
reboot
```

Verify:

```bash
dmesg | grep IOMMU
```

### 2. Isolate the GPU (and its audio function) from the host

Identify the GPU's PCI IDs:

```bash
lspci -nnv | grep -i vga -A 1
```

Note both the GPU (e.g. `0000:01:00.0`) and its associated audio function (e.g. `0000:01:00.1`) —
GPUs almost always have a paired audio device that must be passed through together.

Blacklist the host's own GPU driver in `/etc/modprobe.d/blacklist.conf`:

```bash
blacklist nouveau
blacklist nvidiafb
blacklist radeon
```

Bind both device functions to `vfio-pci` in `/etc/modprobe.d/vfio.conf`:

```bash
options vfio-pci ids=0000:<GPU_ID>,0000:<AUDIO_ID>
```

```bash
update-initramfs -u -k all
reboot
```

Verify the device is now bound to `vfio-pci` instead of its normal driver:

```bash
lspci -nnk | grep -A 3 'VGA'
```

### 3. Install host drivers (NVIDIA only)

Only needed if the host itself also needs to use the GPU (e.g. for its own transcoding) — for pure
passthrough this step is normally skipped, since the host shouldn't touch the device at all once
it's bound to `vfio-pci`.

```bash
apt update && apt install nvidia-driver nvidia-dkms nvidia-headless-no-dkms
reboot
```

### 4. Assign the GPU to a VM

In the Proxmox web UI: select the VM → **Hardware → Add → PCI Device** → pick the GPU → enable
**All Functions** (binds the GPU and its audio device together) and, if it should be the guest's
primary display, **Primary GPU**. Save and start the VM.

## Advanced

- **Above 4G Decoding / Resizable BAR** (BIOS): can improve performance for modern GPUs once
  passthrough is otherwise working.
- **vGPU / mediated-device passthrough**: NVIDIA enterprise GPUs (e.g. Tesla-class) with vGPU
  licensing, or Intel iGPUs with SR-IOV, can split one physical GPU across multiple VMs instead of
  handing it to one VM exclusively.
- **Error 43 (NVIDIA consumer GPUs in Windows guests)**: the driver detects it's running in a VM
  and refuses to initialize. Workaround — add to the VM's config
  (`/etc/pve/qemu-server/<VMID>.conf`):
  ```conf
  args: -cpu host,vendor_id=FakeVendorID
  ```
  Patched drivers or unlock scripts exist as alternatives but sit in a legal gray area — prefer the
  vendor-ID spoofing approach above where it's sufficient.

## Troubleshooting

- **Check IOMMU groups** — a device sharing a group with something else the host needs can block
  passthrough of just that one device:
  ```bash
  find /sys/kernel/iommu_groups/ -type l
  ```
- **`dmesg`** is the primary source for driver bind/unbind and VFIO errors.
- **AMD GPUs**: watch for "reset bugs" (the GPU failing to reset cleanly when a VM stops/restarts)
  — usually fixed by a firmware update, occasionally needs a vendor-specific reset workaround.

## Related

- [PCI address drift breaking hypervisor
  passthrough](pci-address-drift-passthrough.md) — a separate failure mode where a passthrough
  config that pins a device by raw PCI bus address breaks after that address gets reassigned to a
  different device across a reboot; a real risk once passthrough is set up this way.

## Sources

- [Legacy wiki.js: Proxmox GPU passthrough for VMs (private)](../../../../../sources/administration/proxmox/2026-09-28-gpu-passthrough-setup.md)
