---
type: guide
title: Passing a GPU through to a TrueNAS Scale app
description: Select a GPU for a TrueNAS Scale app (no PCI passthrough/isolation needed) and work around the 24.10 GPU-selection bug via midclt.
tags: [truenas, gpu, nvidia, jellyfin, docker, apps]
status: draft
resource:
created: 2026-09-28T18:54:09Z
updated: 2026-09-28T18:54:09Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:54:09Z
verified: []
stale_after: 2027-09-28T18:54:09Z
sources:
- id: 2026-09-28-gpu-passthrough-app-selection
  resource: 'private:/sources/administration/truenas/2026-09-28-gpu-passthrough-app-selection.md'
relations: []
superseded_by:
---

# Passing a GPU through to a TrueNAS Scale app

TrueNAS Scale can hand a system's GPU to an individual app (e.g. Jellyfin or Plex for hardware
transcoding) without full PCI passthrough or GPU isolation — those mechanisms are for a different
scenario (dedicating the GPU exclusively to a VM). This guide covers the simple app-level GPU
selection and a workaround for a TrueNAS Scale 24.10 bug that could break it.

## Prerequisites

- TrueNAS Scale 22.12 or later.
- A GPU suitable for the workload (NVIDIA or AMD) already installed and, if the host is itself a
  VM, passed through to it at the hypervisor level (e.g. Proxmox PCI passthrough) with a working
  driver bound inside the guest.
- The app to be configured already installed.

## Steps

### 1. Select the GPU in the app's settings

TrueNAS Scale ships with the necessary GPU drivers preinstalled. Open the target app's
**Resources/GPU Configuration** settings and select the desired GPU directly.

**Do not** enable general PCI passthrough or GPU isolation for this — select only the GPU in the
app's own resource settings. Those other options are for handing the GPU to a VM instead, and are
unrelated to (and conflict with) simple per-app GPU use.

### 2. Workaround for the TrueNAS Scale 24.10 GPU-selection bug

TrueNAS Scale 24.10 has a known bug where the GPU cannot be selected for an app through the UI at
all, failing with:

```
Expected [uuid] to be set for GPU in slot [0000:02:00.0] in
```

Work around it via the `midclt` CLI instead of the UI.

Identify the available GPU(s) and their UUIDs/PCI slots:

```bash
midclt call app.gpu_choices | jq
```

Apply the GPU assignment directly to the app's config:

```bash
midclt call -j app.update <app-name> '{"values": {"resources": {"gpus": {"use_all_gpus": false, "nvidia_gpu_selection": {"<pci-slot>": {"use_gpu": true, "uuid": "<gpu-uuid>"}}}}}}'
```

For example (illustrative values — substitute the app name, PCI slot and UUID from the previous
step's output):

```bash
midclt call -j app.update jellyfin '{"values": {"resources": {"gpus": {"use_all_gpus": false, "nvidia_gpu_selection": {"0000:02:00.0": {"use_gpu": true, "uuid": "GPU-7cbffdba-cee8-ab11-5d68-987dd76c57e1"}}}}}}'
```

## Verify

Reopen the app's resource settings in the UI — the GPU should now show as assigned. Check the
app's own hardware-acceleration status (e.g. Jellyfin's Dashboard → Playback, or `nvidia-smi`
inside the app's container if shell access is available) to confirm it's actually being used.

## Troubleshooting

- **GPU selection fails with `Expected [uuid] to be set for GPU in slot ...`**: this is the known
  24.10 bug — use the `midclt` workaround above instead of the UI picker.
- **GPU doesn't appear in `app.gpu_choices` at all**: this points to a driver/passthrough problem
  one layer below app-level selection — check that the GPU's driver is actually bound inside
  TrueNAS itself (e.g. `nvidia-smi` at the host shell) before troubleshooting the app config. A
  GPU architecture unsupported by the currently bundled driver (for example, Pascal-generation
  NVIDIA cards under a version that only ships the open-source kernel module) will not show up
  here no matter what app-level settings are tried — see [NVIDIA open kernel module dropping
  Pascal GPU support](../../../concepts/administration/truenas/nvidia-open-driver-pascal-support.md).

## Related

- [NVIDIA open kernel module dropping Pascal GPU support](../../../concepts/administration/truenas/nvidia-open-driver-pascal-support.md) — a driver-level GPU recognition failure that sits one layer below the app-level selection this guide covers.

## Sources

- [Legacy wiki.js (de, translated): TrueNAS Scale GPU passthrough for apps](../../../../../sources/administration/truenas/2026-09-28-gpu-passthrough-app-selection.md) — private source; German-only wiki.js page, no English original existed
- [TrueNAS forums: Docker apps and UUID issue with NVIDIA GPU after upgrade to 24.10](https://forums.truenas.com/t/docker-apps-and-uuid-issue-with-nvidia-gpu-after-upgrade-to-24-10/22547)
