---
type: guide
title: Removing the Proxmox VE subscription notice
description: Patch the community/no-subscription Proxmox VE web UI to stop showing the 'no valid subscription' login dialog, and how to revert it.
tags: [proxmox, subscription, web-ui]
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
- id: 2026-09-28-proxmox-helper-tweaks
  resource: 'private:/sources/administration/proxmox/2026-09-28-proxmox-helper-tweaks.md'
relations: []
superseded_by:
---

# Removing the Proxmox VE subscription notice

Proxmox VE's web UI shows a "No valid subscription" dialog on every login when the node has no
paid subscription key registered. This is a cosmetic nag, not a functional limitation — the
following patch suppresses it. Tested up to Proxmox VE 8.3; reapply after any update that touches
the `proxmox-widget-toolkit` package.

![The "No valid subscription" login dialog this patch
suppresses](removing-proxmox-subscription-notice/subscription-notice-dialog.png)

## Prerequisites

- SSH or web-UI shell access to the node as root.

## Steps

### 1. Patch `proxmoxlib.js`

```sh
sed -Ezi.bak "s/(function\(orig_cmd\) \{)/\1\n\torig_cmd();\n\treturn;/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js && systemctl restart pveproxy.service
```

This edits the function that normally checks subscription status before running the originally
requested UI operation, making it run the operation immediately and return, skipping the check
entirely. A `.bak` backup of the original file is created automatically (the `-i.bak` flag).

### 2. Clear your browser cache

The web UI caches `proxmoxlib.js` client-side — open a new tab or restart the browser (or hard
reload) for the change to take effect.

## Verify

Log into the Proxmox web UI again — the subscription dialog should no longer appear.

## Troubleshooting

- **Notice reappears after an update**: any update that reinstalls `proxmox-widget-toolkit`
  overwrites the patched file — reapply the one-liner.

## Reverting

Three ways to undo the change:

1. Restore the automatic backup:
   ```bash
   mv /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js.bak /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
   ```
2. Reinstall the package fresh from the repository:
   ```bash
   apt-get install --reinstall proxmox-widget-toolkit
   ```
3. Manually re-edit `proxmoxlib.js` to remove the inserted lines.

## Related

- [Reducing swap usage on a hypervisor host to protect SSD lifespan](tuning-swappiness.md) — another
  small generic Proxmox VE host tweak from the same source.

## Sources

- [Legacy wiki.js: Proxmox helper tweaks (private)](../../../../../sources/administration/proxmox/2026-09-28-proxmox-helper-tweaks.md)
