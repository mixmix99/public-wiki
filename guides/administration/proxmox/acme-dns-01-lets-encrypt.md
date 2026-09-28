---
type: guide
title: Automatic Let's Encrypt certificates on a hypervisor via ACME DNS-01
description: Set up automatic Let's Encrypt certificates on Proxmox VE nodes via the ACME DNS-01 challenge with a DNS provider plugin, including the config-encoding workaround for credentials containing newlines.
tags: [proxmox, acme, letsencrypt, dns, tls, certificate]
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
- id: 2026-09-28-proxmox-acme-dns-letsencrypt
  resource: 'private:/sources/administration/proxmox/2026-09-28-proxmox-acme-dns-letsencrypt.md'
relations: []
superseded_by:
---

# Automatic Let's Encrypt certificates on a hypervisor via ACME DNS-01

Proxmox VE has built-in ACME support for issuing Let's Encrypt TLS certificates for its web UI.
Using the **DNS-01 challenge** instead of the HTTP-01 challenge means the node doesn't need to be
publicly reachable on port 80/443 — only the DNS provider's API needs to be reachable from the
node, which makes it the right choice for hypervisors on a private/internal network.

## How it works

1. Proxmox places an ACME order with Let's Encrypt for the configured domain.
2. Let's Encrypt responds with a DNS-01 challenge token.
3. Proxmox calls the configured DNS plugin (`acme.sh` under the hood) to create a
   `_acme-challenge.<domain>` TXT record via the provider's API.
4. Let's Encrypt validates the TXT record.
5. The plugin removes the TXT record and Proxmox installs the issued certificate on `pveproxy`.
6. Proxmox handles automatic renewal (~30 days before expiry) via a systemd timer — no manual
   action needed in normal operation.

DNS plugins live at `/usr/share/proxmox-acme/dnsapi/` on the node — one shell script per supported
provider.

## Prerequisites

- A domain pointed at your DNS provider (e.g. `myhost.<domain>`).
- API credentials from that DNS provider.
- An ACME account registered in Proxmox (one shared account covers every node in a cluster).

## Steps

### 1. Register (or check) an ACME account

Only needed once per cluster:

```bash
pvenode acme account register default your@email.com
```

Check existing accounts:

```bash
pvenode acme account list
pvenode acme account info default
```

### 2. Add the DNS plugin

Proxmox stores plugin configs in `/etc/pve/priv/acme/plugins.cfg` — cluster-wide, synced by
`pmxcfs`. The `data` field holding API credentials is stored **base64url-encoded**, and the
`pvesh create /cluster/acme/plugins` CLI command rejects `--data` values containing a literal
newline. Since most providers' credential blocks are multi-line (`KEY=value` pairs joined by
`\n`), write the config file directly instead of going through the CLI:

```bash
perl -e '
use MIME::Base64 qw(encode_base64url);
my $raw = "PROVIDER_KEY1=<your-key1>\nPROVIDER_KEY2=<your-key2>";
my $enc = encode_base64url($raw);
$enc =~ s/\n//g;
my $cfg = "standalone: standalone\n\ndns: <plugin-name>\n\tapi <plugin-name>\n\tdata $enc\n";
open(my $fh, ">", "/etc/pve/priv/acme/plugins.cfg") or die $!;
print $fh $cfg;
close $fh;
'
```

Run this on **any one cluster node** — the file is synced cluster-wide.

Each DNS plugin expects different environment-variable names for its credentials (e.g.
`CLOUDFLARE_Token`, `HETZNER_Token`, or a `<PROVIDER>_PREFIX`/`<PROVIDER>_SECRET` pair for
API-key/secret style providers). Check if a plugin exists for your provider and read its header
comments for the exact variable names:

```bash
ls /usr/share/proxmox-acme/dnsapi/ | grep <provider>
```

> **Why Perl, not the API?** `pvesh create /cluster/acme/plugins` refuses input with embedded
> newlines, making a direct API call impractical for any multi-line credential block. Writing the
> already-base64url-encoded value straight into the config file sidesteps that CLI limitation
> entirely, and `pmxcfs` still syncs it cluster-wide like any other write to `/etc/pve/`.

### 3. Configure the domain on each node

This is a **per-node** setting — repeat it on every node individually, even though the plugin
config itself is cluster-wide:

```bash
pvenode config set --acmedomain0 <hostname>.<domain>,plugin=<plugin-name>
```

Verify:

```bash
pvenode config get | grep acmedomain
```

### 4. Order the certificate

```bash
pvenode acme cert order
```

This places the ACME order, creates the DNS TXT record, waits for propagation (default 30 s),
validates, downloads the cert, installs it on `pveproxy`, and restarts `pveproxy`. Takes roughly
60–90 seconds end to end.

## Verify

```bash
systemctl status pvenode-acme-update.timer   # confirms the renewal timer is active
pvenode acme cert renew                      # force a manual renewal to test the full flow
```

## Troubleshooting

- **"You didn't specify an API prefix/secret yet"** (or similar provider-specific message): the
  `data` field in `plugins.cfg` isn't correctly base64url-encoded, so the raw content handed to
  `proxmox-acme` is malformed — re-run the Perl one-liner from step 2, double-checking the
  variable names against the plugin's own header comments.
- **"property contains a line feed"**: you tried to pass a multi-line credential block through
  `pvesh --data` directly — use the file-write workaround instead, not the API.
- **TXT record not found / validation fails**: DNS propagation can be slower than the default
  30-second wait. Increase the plugin's validation delay:
  ```bash
  pvesh set /cluster/acme/plugins/<plugin-name> --validation-delay 120
  ```

## Related

<None yet.>

## Sources

- [Legacy wiki.js: Proxmox ACME Let's Encrypt DNS challenge (private)](../../../../../sources/administration/proxmox/2026-09-28-proxmox-acme-dns-letsencrypt.md)
