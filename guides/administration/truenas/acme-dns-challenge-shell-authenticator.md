---
type: guide
title: TrueNAS Scale ACME certificates via DNS-01 with a shell authenticator
description: Set up automatic Let's Encrypt TLS certificates on TrueNAS Scale using the shell DNS authenticator plugin for any DNS provider without built-in support.
tags: [truenas, acme, letsencrypt, dns, tls, certificate]
status: draft
resource:
created: 2026-09-28T17:02:31Z
updated: 2026-09-28T17:02:31Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:02:31Z
verified: []
stale_after: 2027-09-28T17:02:31Z
sources:
- id: 2026-09-28-acme-dns-shell-authenticator
  resource: 'private:/sources/administration/truenas/2026-09-28-acme-dns-shell-authenticator.md'
relations: []
superseded_by:
---

# TrueNAS Scale ACME certificates via DNS-01 with a shell authenticator

TrueNAS Scale has built-in ACME support for Let's Encrypt certificates. Since a home-lab TrueNAS
box usually doesn't need to be publicly reachable, the DNS-01 challenge is the right method — no
inbound port-80/443 exposure needed. TrueNAS ships built-in DNS providers (Cloudflare,
DigitalOcean, Route53, OVH, ...); for any other provider it doesn't natively support, the **shell
authenticator** calls a custom script instead.

## How it works

1. TrueNAS places an ACME order with Let's Encrypt for the configured domain.
2. Let's Encrypt issues a DNS-01 challenge.
3. TrueNAS calls the shell script: `script set <domain> <validation_name> <txt_value>`.
4. The script creates the `_acme-challenge.<domain>` TXT record via the provider's API.
5. After a propagation delay, TrueNAS triggers validation, then calls `script unset ...` to clean
   up the TXT record.
6. The certificate is issued and can be set as the GUI certificate.
7. TrueNAS handles automatic renewal at `renew_days` before expiry — no ongoing action needed.

## Prerequisites

- A domain pointed at your DNS provider (e.g. `<hostname>.<domain>`).
- API credentials for that DNS provider.
- The authenticator script must reside **within a TrueNAS pool/dataset** — TrueNAS validates this
  and rejects paths outside `/mnt/`.

## Steps

### 1. Write the shell authenticator script

The script lives on a TrueNAS dataset (e.g. `/mnt/<pool>/scripts/`). TrueNAS calls it as:

```
script set   <domain> <validation_name> <txt_value>
script unset <domain> <validation_name> <txt_value>
```

The shape below (example against a DNS provider whose classic API doesn't have a built-in TrueNAS
authenticator) generalizes to any provider with a TXT-record REST API — swap the API base URL,
auth header and JSON payloads for your own provider:

```bash
#!/bin/bash
# <provider> DNS authenticator for TrueNAS Scale ACME shell plugin
# Called as: script set|unset <domain> <validation_name> <txt_value>

ACTION="$1"
DOMAIN="$2"
VALIDATION_NAME="$3"
VALIDATION_CONTENT="$4"

API_KEY="<api-key>"                 # store the real value in your password manager, not here
API_BASE="https://api.<provider>.example/dns/v1"

log() { echo "[<provider>-acme] $*" >&2; }

find_zone_id() {
    local record_name="$1"
    local zones
    zones=$(curl -sf -H "X-API-Key: $API_KEY" "$API_BASE/zones") || { log "Failed to list zones"; exit 1; }
    echo "$zones" | python3 -c "
import sys, json
zones = json.load(sys.stdin)
name = '$record_name'.rstrip('.')
best_id = ''
best_len = 0
for z in zones:
    zname = z['name'].rstrip('.')
    if name == zname or name.endswith('.' + zname):
        if len(zname) > best_len:
            best_id = z['id']
            best_len = len(zname)
print(best_id)
"
}

ZONE_ID=$(find_zone_id "$VALIDATION_NAME")
if [ -z "$ZONE_ID" ]; then
    log "Could not find zone for $VALIDATION_NAME"
    exit 1
fi

if [ "$ACTION" = "set" ]; then
    curl -sf -X POST -H "X-API-Key: $API_KEY" -H "Content-Type: application/json" \
        -d "[{\"name\": \"${VALIDATION_NAME}.\", \"type\": \"TXT\", \"content\": \"\\\"${VALIDATION_CONTENT}\\\"\", \"ttl\": 60, \"disabled\": false}]" \
        "$API_BASE/zones/$ZONE_ID/records" > /dev/null
    log "TXT record created"
elif [ "$ACTION" = "unset" ]; then
    RECORDS=$(curl -sf -H "X-API-Key: $API_KEY" "$API_BASE/zones/$ZONE_ID") || { log "Failed to fetch zone"; exit 1; }
    RECORD_ID=$(echo "$RECORDS" | python3 -c "
import sys, json
data = json.load(sys.stdin)
vname = '$VALIDATION_NAME'.rstrip('.')
vcontent = '$VALIDATION_CONTENT'
for r in data.get('records', []):
    if r.get('type') == 'TXT' and r.get('name','').rstrip('.') == vname and r.get('content','').strip('\"') == vcontent:
        print(r['id']); break
")
    if [ -n "$RECORD_ID" ]; then
        curl -sf -X DELETE -H "X-API-Key: $API_KEY" "$API_BASE/zones/$ZONE_ID/records/$RECORD_ID"
        log "Record deleted"
    else
        log "TXT record not found for deletion (may already be gone)"
    fi
fi
```

Save it (e.g. `/mnt/<pool>/scripts/<provider>-acme.sh`) and make it executable:

```bash
chmod 755 /mnt/<pool>/scripts/<provider>-acme.sh
```

Test manually before wiring it into ACME:

```bash
bash /mnt/<pool>/scripts/<provider>-acme.sh set example.com _acme-challenge.example.com testtoken
# verify the TXT record appears in DNS, then:
bash /mnt/<pool>/scripts/<provider>-acme.sh unset example.com _acme-challenge.example.com testtoken
```

### 2. Create the DNS authenticator

```bash
midclt call acme.dns.authenticator.create '{
  "name": "<provider>",
  "attributes": {
    "authenticator": "shell",
    "script": "/mnt/<pool>/scripts/<provider>-acme.sh",
    "user": "nobody",
    "timeout": 120,
    "delay": 60
  }
}'
```

Note the returned `id` — needed in step 4.

### 3. Create a CSR

```bash
midclt call certificate.create '{
  "name": "<hostname>-csr",
  "create_type": "CERTIFICATE_CREATE_CSR",
  "san": ["<hostname>.<domain>"],
  "common": "<hostname>.<domain>",
  "email": "<your-email>",
  "country": "<country>",
  "state": "<state>",
  "city": "<city>",
  "organization": "<org>",
  "organizational_unit": "<org>",
  "digest_algorithm": "SHA256",
  "key_type": "RSA",
  "key_length": 2048
}'
```

Find the CSR ID:

```bash
midclt call certificate.query | python3 -c "
import sys,json; certs=json.load(sys.stdin)
[print(c['id'], c['name']) for c in certs if c.get('cert_type_CSR')]
"
```

### 4. Issue the ACME certificate

Replace `<csr_id>` and `<authenticator_id>` with the IDs from steps 2–3:

```bash
JOB=$(midclt call certificate.create '{
  "name": "<hostname>-cert",
  "create_type": "CERTIFICATE_CREATE_ACME",
  "csr_id": <csr_id>,
  "acme_directory_uri": "https://acme-v02.api.letsencrypt.org/directory",
  "tos": true,
  "dns_mapping": {"<hostname>.<domain>": <authenticator_id>},
  "renew_days": 30
}')

# Monitor progress
watch -n5 "midclt call core.get_jobs \"[[\\\"id\\\",\\\"=\\\",$JOB]]\" | python3 -c \"import sys,json; j=json.load(sys.stdin)[0]; print(j['state'], j.get('progress',{}).get('description',''), (j.get('error') or '')[:200])\""
```

### 5. Set as the GUI certificate

```bash
CERT_ID=$(midclt call certificate.query | python3 -c "
import sys,json; certs=json.load(sys.stdin)
[print(c['id']) for c in certs if c['name'] == '<hostname>-cert']
")

midclt call system.general.update "{\"ui_certificate\": $CERT_ID}"
midclt call system.general.ui_restart
```

## Verify

The TrueNAS web UI should now serve a valid Let's Encrypt certificate for `<hostname>.<domain>`.
Check expiry any time with:

```bash
midclt call certificate.query | python3 -c "
import sys,json; certs=json.load(sys.stdin)
[print(c['name'], c.get('until')) for c in certs if not c.get('cert_type_CSR')]
"
```

## Troubleshooting

- **Validation keeps failing / timing out:** increase the authenticator's `delay` (default 60s) —
  this is the propagation wait before Let's Encrypt is asked to validate the TXT record.
- **Script rejected at creation time:** the script path must be inside a pool dataset (`/mnt/...`)
  — TrueNAS refuses authenticator scripts outside it.
- **Provider's "search/filter" API endpoint returns nothing:** some providers' suffix/type filter
  query parameters are unreliable; fetch the whole zone and filter for the matching record in the
  script itself instead of relying on server-side filtering.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: TrueNAS Scale ACME DNS-01 shell authenticator](../../../../../sources/administration/truenas/2026-09-28-acme-dns-shell-authenticator.md) — private source (original setup notes, including the real provider used in this home lab)
