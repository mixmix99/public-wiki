---
type: guide
title: Lower idle power with the amd-pstate-epp balance_power profile
description: Persistently switch an AMD host from the performance profile to balance_power with a systemd unit and measure the saving with RAPL.
tags:
- linux
- amd
- power
- cpufreq
- amd-pstate
- systemd
status: draft
resource:
created: 2026-11-08T15:18:53Z
updated: 2026-11-08T15:18:53Z
generated:
  by: claude/sonnet-5.5
  at: 2026-11-08T15:18:53Z
verified: []
stale_after: 2027-11-08T15:18:53Z
sources:
- id: 2026-11-08-homelab-power-audit
  resource: private:/sources/administration/servers/2026-11-08-homelab-power-audit.md
relations: []
superseded_by:
---

# Lower idle power with the amd-pstate-epp balance_power profile

Some distributions and hypervisors leave AMD CPUs on the `performance` governor with the
`performance` energy preference. This guide switches an always-on server to `balance_power`,
makes it survive reboots with a small systemd unit, and measures the saving with the CPU's own
power counter. On a lightly loaded Ryzen 9 5950X host this saved about 7 W (about 49 W to 42 W).
At EUR 0.28 per kWh, 10 W of continuous draw costs about EUR 25 per year.

## Prerequisites

- An AMD CPU using the `amd-pstate-epp` driver in active mode. Check:
  `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver` should print `amd-pstate-epp`.
- Root access on the host, and a host that is mostly idle (under constant full load the profile
  makes no difference).
- A readable RAPL counter for measuring: `/sys/class/powercap/intel-rapl:0/energy_uj` (the
  directory is named `intel-rapl` even on AMD Zen).

## Steps

### 1. Check the current settings

```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor               # e.g. performance
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference  # e.g. performance
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_available_preferences
```

### 2. Measure a baseline

Take several 10 s samples; one busy sample can distort a short test. The counter wraps, so
handle `b < a`.

```bash
R=/sys/class/powercap/intel-rapl:0; MAX=$(cat $R/max_energy_range_uj)
for i in 1 2 3 4; do
  a=$(cat $R/energy_uj); sleep 10; b=$(cat $R/energy_uj)
  [ "$b" -lt "$a" ] && b=$((b+MAX))
  echo "$(( (b-a)/10000000 )) W (approx)  load=$(cut -d' ' -f1 /proc/loadavg)"
done
```

### 3. Try it at runtime first (not persistent)

On `amd-pstate-epp` the energy preference can only be changed while the governor is `powersave`:

```bash
for c in /sys/devices/system/cpu/cpu[0-9]*/cpufreq; do
  echo powersave > "$c/scaling_governor"
  echo balance_power > "$c/energy_performance_preference"
done
```

Repeat the measurement. For a trustworthy result alternate the profiles (A-B-A-B-A) and compare
medians: spikes from other workloads show up under both profiles. To go back before persisting,
write `performance` to both files, or reboot.

### 4. Make it persistent

```bash
cat > /usr/local/sbin/cpu-power-profile.sh <<'EOF'
#!/bin/sh
# Governor powersave + EPP balance_power on all CPUs (EPP is only writable under powersave).
for c in /sys/devices/system/cpu/cpu[0-9]*/cpufreq; do
  echo powersave > "$c/scaling_governor"
  echo balance_power > "$c/energy_performance_preference"
done
EOF
chmod 755 /usr/local/sbin/cpu-power-profile.sh

cat > /etc/systemd/system/cpu-power-profile.service <<'EOF'
[Unit]
Description=CPU power profile (amd-pstate-epp balance_power) to save idle power
After=sysinit.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/cpu-power-profile.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now cpu-power-profile.service
```

## Verify

```bash
systemctl is-enabled cpu-power-profile.service; systemctl is-active cpu-power-profile.service
cat /sys/devices/system/cpu/cpu[0-9]*/cpufreq/scaling_governor | sort | uniq -c            # all powersave
cat /sys/devices/system/cpu/cpu[0-9]*/cpufreq/energy_performance_preference | sort | uniq -c  # all balance_power
```

After the next reboot, confirm `energy_performance_preference` still says `balance_power`.

## Troubleshooting

- **Writing the preference fails or has no effect:** the governor is still `performance`. Set
  `powersave` first, as in the script.
- **Negative or absurd watt values:** the RAPL counter wrapped between samples (add
  `max_energy_range_uj`), or the arithmetic is off. Energy is in microjoules: watts =
  (delta in uJ) / (seconds x 1,000,000).
- **No saving measured:** the host is probably under constant load (for example a transcoder
  using all cores). The profile helps at idle and partial load only.
- **Unit listed but a different tool resets the profile:** check for `tuned`, `power-profiles-daemon`
  or `cpupower` services (`systemctl list-unit-files | grep -E "cpupower|tuned|power-profiles"`).
- **Undo:** `systemctl disable --now cpu-power-profile.service`, then write `performance` to the
  governor and the preference, or reboot.
- Measured values here come from one 16-core and one 6-core host and short samples; treat the
  saving as an estimate for your own hardware.

## Related

- [Bogus MCE hardware-error events on AMD Zen 3 with Linux 7.0](../../../concepts/administration/linux/amd-zen3-bogus-mce-events-linux-7.md)
  (same CPU family and kernel generation, different topic)

## Sources

- [Original notes (private)](../../../../../sources/administration/servers/2026-11-08-homelab-power-audit.md)
