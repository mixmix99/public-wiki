---
type: guide
title: Using a transactional email API as an SMTP relay for self-hosted services
description: Point self-hosted apps' SMTP settings at a free-tier transactional email provider (e.g. Brevo) instead of running your own mail server, including per-app config-persistence gotchas.
tags:
- email
- smtp
- brevo
- self-hosted
- docker
- dkim
status: draft
resource:
created: 2026-09-27T19:49:16Z
updated: 2026-09-28T17:02:28Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:02:28Z
verified: []
stale_after: 2027-09-27T19:49:16Z
sources:
- id: 2026-09-27-brevo-smtp-relay-setup
  resource: private:/sources/administration/services/2026-09-27-brevo-smtp-relay-setup.md
- id: 2026-09-28-transactional-email-relay-concept
  resource: private:/sources/administration/services/2026-09-28-transactional-email-relay-concept.md
relations: []
superseded_by:
---

# Using a transactional email API as an SMTP relay for self-hosted services

Self-hosted apps (password managers, Git forges, monitoring tools, media managers, task
managers...) routinely want to send email — password resets, notifications, alerts — but running
your own outbound mail server is its own project (reverse DNS, SPF/DKIM/DMARC, IP reputation,
getting deliverability past spam filters). A free-tier transactional email provider (Brevo,
Mailgun, SendGrid, Postmark, etc.) gives you an authenticated SMTP relay in minutes, which every
one of these apps can point at instead.

## Why not just send email directly from the server

1. **Outbound port 25 is blocked by default** on most cloud providers to curb spam. Getting it
   unblocked requires a manual support request.
2. **IP reputation.** Cloud provider IP ranges are shared pools with a history — a fresh VM's IP
   may already be flagged by spam blocklists from a previous tenant, before you've sent a single
   email.
3. **Missing infrastructure.** Reliable delivery to Gmail/Outlook/etc. requires a matching reverse
   DNS (PTR) record, and properly configured SPF, DKIM, and DMARC records — none of which exist by
   default on a new server or domain.
4. **Low volume, no history.** Occasional transactional email (a handful of messages a day) never
   builds up enough sending reputation with major providers to reliably land in the inbox instead
   of spam.

For a handful of notification/transactional emails, running your own mail server is a lot of
ongoing maintenance for a fragile result — handing it to a relay that already has established IP
reputation and handles the SPF/DKIM/DMARC complexity is far less work.

## How it works

1. Sign up with a provider and verify ownership of a domain you control — typically via DNS
   records (DKIM CNAMEs, sometimes SPF/DMARC too).
2. The provider gives you an SMTP host, port, username and password. This is one shared credential
   pair — you don't need a separate relay account per app.
3. Point each self-hosted app's SMTP settings (host, port, username, password, `From:` address) at
   the relay. Any app that speaks standard SMTP works, regardless of language/stack.
4. Pick a distinct `From:` address per app (e.g. `app1@<domain>`, `app2@<domain>`) so notifications
   stay identifiable, even though they all go through the same relay account.

Typical settings for this pattern (values vary by provider):

| Setting | Typical value |
|---|---|
| SMTP host | `smtp-relay.<provider>.com` |
| Port | `587` |
| Security | STARTTLS |
| Auth | username + password (or an API key used as the password) |

## When to use it / trade-offs

- **Use it when:** you have several self-hosted apps that each need to send a modest volume of
  transactional/notification email (password resets, alerts, reminders), and you don't want the
  operational burden of a real outbound mail server.
- **Free tiers are typically capped** (e.g. a few hundred emails/day) — fine for personal/small
  self-hosted use, not for bulk mail. Check the provider's limits before committing to it for
  anything higher-volume.
- **A naming convention matters once you have several apps on several hosts.** If every host runs
  one app, `<app>@<domain>` is enough. Once a host runs multiple notification-capable apps, prefix
  with the host too (`<host>-<app>@<domain>`) so you can tell at a glance which host a
  notification came from — worth deciding up front rather than retrofitting later.
- **This also covers host-level system mail**, not just app containers: a host's local MTA (e.g.
  `exim4`, `postfix`) can be pointed at the same relay in "smarthost" mode, so cron/apt/system
  mail that would otherwise sit undelivered in a local mailbox (or not go anywhere at all if no
  MTA is installed) actually gets delivered.

## Steps

### 1. Verify your domain with the provider

Add the DKIM CNAME records the provider gives you to your domain's DNS. **Some DNS providers
silently append your zone name to a CNAME target unless you explicitly end the target with a
trailing dot (`.`)** — this breaks verification without any obvious error; if verification hangs,
check your zone's raw record values, not just what the DNS provider's UI displays.

### 2. Point each app at the relay

Most self-hosted apps expose SMTP settings as either environment variables or a config file field
set. Common patterns:

```
SMTP_HOST=smtp-relay.<provider>.com
SMTP_PORT=587
SMTP_USERNAME=<relay-username>
SMTP_PASSWORD=<relay-password>
SMTP_FROM=<app>@<domain>
```

**Watch for apps that persist SMTP config to a file on first run**, independent of their
container's environment variables — a common pattern in self-hosted apps that support
in-app admin settings. Once that file exists, it silently takes priority: editing the compose
file's env vars afterward has no effect until you either edit the persisted file directly or use
the app's own admin UI to change the setting. If email suddenly "stops updating" after a compose
edit, check the app's logs for a startup warning listing which env vars are being overridden by a
persisted config file — this is often exactly what's happening.

### 3. Fix host-level system mail (optional)

If a host's own cron/apt/system mail has nowhere to go (no MTA installed, or an MTA configured for
local delivery only), configure it in smarthost mode against the same relay:

```
# exim4 example: /etc/exim4/update-exim4.conf.conf
dc_eximconfig_configtype='smarthost'
dc_smarthost='smtp-relay.<provider>.com::587'
```

```
# /etc/exim4/passwd.client
smtp-relay.<provider>.com:<relay-username>:<relay-password>
```

Set `/etc/mailname` to your verified domain, and add a sender-rewrite rule so mail appears from a
recognizable address instead of `root@<domain>`:

```
# begin rewrite section of exim4.conf.template
root@<domain>   <host>@<domain>   Eh
```

Apply and verify:

```bash
update-exim4.conf && systemctl restart exim4
echo "test" | mail -s "subject" <your-address>
# then check the mail log for a 250 OK response from the relay
```

The same pattern (smarthost + client-auth file + mailname + sender rewrite) applies to any host
that needs local system mail actually delivered — reuse the same relay credentials, just change
the rewrite target address per host.

## Verify

A standalone test independent of any app's own SMTP test feature confirms the relay itself works:

```python
import smtplib, ssl
from email.mime.text import MIMEText

msg = MIMEText("Test email.")
msg["Subject"] = "SMTP relay test"
msg["From"] = "<app>@<domain>"
msg["To"] = "<your-address>"

server = smtplib.SMTP("smtp-relay.<provider>.com", 587, timeout=15)
server.starttls(context=ssl.create_default_context())
server.login("<relay-username>", "<relay-password>")
server.sendmail(msg["From"], [msg["To"]], msg.as_string())
server.quit()
```

## Troubleshooting

- **Compose-file SMTP env var changes have no effect:** check whether the app persists SMTP
  config to a file on first run (see step 2) — this is the single most common gotcha across
  different self-hosted apps.
- **A connection-string-style SMTP setting (e.g. `smtp://user:pass@host:port`) fails
  authentication when the username contains `@`:** percent-encode the `@` (`%40`) within the URL —
  a raw `@` in the username gets parsed as the userinfo/host separator instead.
- **Migrating from a previous provider:** grep every app's config for the old SMTP host/credentials
  before assuming the migration is complete — it's easy to miss one app still pointed at dead
  credentials from a prior domain/provider setup, silently failing forever with no user-visible
  symptom until someone notices missing notifications.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: Brevo SMTP relay setup](../../../../../sources/administration/services/2026-09-27-brevo-smtp-relay-setup.md) — private source (real configuration)
- [Legacy wiki.js: Sending email via a transactional relay (concept)](../../../../../sources/administration/services/2026-09-28-transactional-email-relay-concept.md) — private source (generic concept, "why not send directly" reasoning)
