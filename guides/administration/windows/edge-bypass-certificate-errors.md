---
type: guide
title: Bypassing a TLS certificate warning in Microsoft Edge
description: The thisisunsafe keyboard shortcut that permanently bypasses a certificate-error interstitial for one site in Microsoft Edge.
tags: [edge, browser, tls, certificate]
status: draft
resource:
created: 2026-09-28T17:06:38Z
updated: 2026-09-28T17:06:38Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:06:38Z
verified: []
stale_after: 2027-09-28T17:06:38Z
sources:
- id: 2026-09-28-hyperv-usb-and-edge-tips
  resource: 'private:/sources/administration/windows/2026-09-28-hyperv-usb-and-edge-tips.md'
relations: []
superseded_by:
---

# Bypassing a TLS certificate warning in Microsoft Edge

Microsoft Edge (and other Chromium-based browsers) has an undocumented keyboard shortcut that
bypasses the certificate-error interstitial page for a site whose TLS certificate it can't
validate — useful for a self-signed cert on an internal device or service you trust and control.

## Steps

### 1. Trigger the interstitial

Open the site with the certificate error in Microsoft Edge.

### 2. Type the bypass phrase

1. Click anywhere on the error page to give it keyboard focus.
2. Type `thisisunsafe` and press Enter — **type it, don't paste it**; pasting does not trigger the
   bypass.

The page loads immediately, and Edge remembers the exception for that site going forward.

## Verify

Reload the site — it should load without the certificate warning reappearing.

## Troubleshooting

- **Nothing happens when typing the phrase:** make sure the error page itself has focus (click on
  it first) and that you're typing, not pasting, the characters.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: Microsoft Edge Tips and Tricks](../../../../../sources/administration/windows/2026-09-28-hyperv-usb-and-edge-tips.md) — private source
