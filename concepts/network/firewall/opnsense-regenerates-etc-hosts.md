---
type: concept
title: OPNsense regenerates /etc/hosts on boot and service reload
description: Why entries a custom script adds to /etc/hosts on OPNsense vanish after a reboot or 'Reload all services', and how to re-apply them with a syshook start script.
tags: [opnsense, dns, hosts-file, syshook]
status: draft
resource:
created: 2026-11-07T13:09:51Z
updated: 2026-11-07T13:09:51Z
generated:
  by: claude/opus-5.5
  at: 2026-11-07T13:09:51Z
verified: []
stale_after: 2028-11-06T13:09:51Z
sources:
- id: 2026-11-07-opnsense-reboot-dns-outage
  resource: private:/sources/network/firewall/2026-11-07-opnsense-reboot-dns-outage.md
relations: []
superseded_by:
---

# OPNsense regenerates /etc/hosts on boot and service reload

On OPNsense, `/etc/hosts` is not a file you own: the core rewrites it from scratch, and anything a
custom script appended to it (for example DHCP lease names for a DNS service that reads the hosts
file) is gone until that script runs again.

## How it works

`system_hosts_generate()` in `/usr/local/etc/inc/system.inc` writes only three kinds of lines —
`localhost` (IPv4/IPv6) and the firewall's own hostname on its primary interface — with
`file_put_contents('/etc/hosts', …)`. It is called from `system_resolver_configure()`, which runs:

- during boot (`rc.bootup`),
- on **Reload all services** (console menu / `rc.reload_all`),
- when DNS-related system settings are applied.

A script that appends a block to `/etc/hosts` therefore loses it on every one of these events. A
DNS server that answers from the hosts file (AdGuard Home does this by default; dnsmasq reads it on
start) then returns NXDOMAIN for those names, which users experience as "the network is dead"
even though routing works.

## When to use it / trade-offs

If you must keep entries in `/etc/hosts`, re-apply them after the core has written the file:

```sh
#!/bin/sh
# /usr/local/etc/rc.syshook.d/start/99-my-hosts  (chmod 755)
# Runs at the end of boot. Background it so boot is not delayed; retry because
# services the script depends on may not be ready yet.
(
    for i in 1 2 3 4 5 6 7 8 9 10 11 12; do
        /usr/local/sbin/configctl <your-action>      # or call your script directly
        grep -q '<marker line your script writes>' /etc/hosts && exit 0
        sleep 10
    done
    logger -t my-hosts "boot hook: hosts entries still missing"
) > /dev/null 2>&1 &
exit 0
```

There is no syshook for *Reload all services*, so also run the script from a frequent cron job
(System → Settings → Cron, a configd action); make it a no-op when nothing changed.

Better long term: don't depend on `/etc/hosts` at all — let the DNS server forward the local domain
to a resolver that knows the leases (e.g. `[/<local-domain>/]<firewall-ip>:<dnsmasq-port>` as an
AdGuard upstream), or use the DHCP server's own DNS registration.

## Pitfalls

- **"Reload all services" makes it worse:** it regenerates `/etc/hosts` again, so using it as a
  "fix" after a reboot just restarts the outage window; recovery comes from the next script run.
- **Cron jobs from the GUI that call configd actions** are written to `/var/cron/tabs/nobody`, not
  `root`'s crontab — look there when checking the schedule.
- Custom files under `rc.syshook.d/` survive firmware upgrades but are **not** in `config.xml`, so a
  config backup/restore does not bring them back.
- Root's login shell on OPNsense is csh: when running multi-line commands over SSH, pipe them to
  `sh -s` to avoid quoting errors.

## Related

- [AdGuard Home on OPNsense (mimugmail)](../../../guides/network/dns/adguard-home-on-opnsense-mimugmail.md) — the plugin setup where the hosts file feeds local `*.local` answers.

## Sources

- [Original session notes (private)](../../../../../sources/network/firewall/2026-11-07-opnsense-reboot-dns-outage.md)
