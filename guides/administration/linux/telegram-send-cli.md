---
type: guide
title: Installing telegram-send for scripted Telegram notifications
description: Install the telegram-send CLI and work around a python-telegram-bot version incompatibility that breaks it on first run.
tags: [linux, telegram, cli, notifications, python]
status: draft
resource:
created: 2026-09-28T18:54:10Z
updated: 2026-09-28T18:54:10Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:54:10Z
verified: []
stale_after: 2027-09-28T18:54:10Z
sources:
- id: 2026-09-28-telegram-send-cli
  resource: 'private:/sources/administration/linux/2026-09-28-telegram-send-cli.md'
relations: []
superseded_by:
---

# Installing telegram-send for scripted Telegram notifications

[telegram-send](https://github.com/rahiel/telegram-send) is a command-line tool for sending
messages to Telegram — useful for scripts, cron jobs, and other automation that needs a simple
notification channel.

## Prerequisites

- Python 3 with `pip3` on the host.

## Steps

### 1. Install telegram-send

```bash
sudo pip3 install telegram-send
```

### 2. Work around a python-telegram-bot incompatibility

Depending on which `python-telegram-bot` version pip resolves, `telegram-send` can fail on first
run with:

```
cannot import name 'MAX_MESSAGE_LENGTH' from 'telegram.constants'
```

This happens when a newer `python-telegram-bot` release removes/renames the constant
`telegram-send`'s (older, unmaintained-at-the-time) code imports directly. Force-reinstall a known
compatible version:

```bash
pip3 install --force-reinstall -v "python-telegram-bot==13.5"
```

This is a version-pin workaround, not a real fix — check whether it's still needed against the
current `telegram-send` release before assuming it applies; a properly updated `telegram-send`
may no longer need the pin, or may need a different one.

## Verify

Run `telegram-send` interactively once to complete its bot-token setup, then send a test message:

```bash
telegram-send "test message"
```

## Troubleshooting

- **`cannot import name 'MAX_MESSAGE_LENGTH' from 'telegram.constants'`**: apply the
  `python-telegram-bot==13.5` pin above.

## Related

<None yet.>

## Sources

- [Legacy wiki.js (de, translated): telegram-send CLI install](../../../../../sources/administration/linux/2026-09-28-telegram-send-cli.md) — private source; German-only wiki.js page, no English original existed
