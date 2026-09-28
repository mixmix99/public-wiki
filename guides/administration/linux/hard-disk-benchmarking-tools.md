---
type: guide
title: Benchmarking Linux disk and SSD speed with hdparm, dd and fio
description: Three common Linux tools for measuring disk/SSD read, write and random-I/O throughput, from a quick built-in check to a realistic load test.
tags: [linux, storage, benchmark, hdparm, dd, fio]
status: draft
resource:
created: 2026-09-28T17:05:57Z
updated: 2026-09-28T17:05:57Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T17:05:57Z
verified: []
stale_after: 2027-09-28T17:05:57Z
sources:
- id: 2026-09-28-storage-and-dns-tools
  resource: 'private:/sources/administration/linux/2026-09-28-storage-and-dns-tools.md'
relations: []
superseded_by:
---

# Benchmarking Linux disk and SSD speed with hdparm, dd and fio

Since a drive is often the bottleneck for server performance, it's worth being able to quickly
measure its actual throughput. Three commonly available tools cover most needs, from a quick
sanity check to a realistic sustained-load test.

## Prerequisites

- Root/sudo access to run the reads/writes below.
- `fio` may need installing; `hdparm` and `dd` are pre-installed on almost every Linux
  distribution.

## Steps

### 1. Quick read speed check with hdparm

```bash
hdparm -tT /dev/<device>
```

`-t` measures raw read speed from the device; `-T` measures read speed from cache. Replace
`/dev/<device>` with the drive to test.

To test write speed:

```bash
hdparm -tT --write-sector /dev/<device>
```

> **Warning:** this test can damage data on the drive. Only run it against a drive whose data you
> don't need.

### 2. Simple sequential throughput with dd

```bash
dd if=/dev/zero of=/tmp/test bs=1G count=10 oflag=direct
```

Copies 10 blocks of 1 GB each from `/dev/zero` to `/tmp/test`. `oflag=direct` bypasses the page
cache so the result reflects real device write speed rather than RAM buffering. Delete the test
file afterward.

### 3. Realistic random I/O load with fio

```bash
apt install fio
```

```bash
fio --name=randwrite --ioengine=libaio --rw=randwrite --bs=4k --size=100G --numjobs=1 --runtime=120s --time_based --group_reporting
```

`--rw=randwrite` runs a random-write pattern (closer to real-world database/VM workloads than
sequential `dd`); `--bs` sets the block size, `--size` the test file size, `--numjobs` the number
of parallel jobs, and `--runtime`/`--time_based` bound the test to a fixed duration rather than a
fixed amount of data.

## Verify

Compare results across runs/drives using the same parameters (block size, size, job count) — raw
numbers are only meaningful relative to a consistent test setup, not as absolute figures to
compare against unrelated benchmarks online.

## Troubleshooting

- **hdparm numbers look too good to be true:** you may be measuring cache (`-T`) rather than the
  device itself (`-t`) — check which flag produced the number.
- **dd numbers are inflated:** confirm `oflag=direct` is present; without it you're partly
  measuring page-cache write-back speed, not the device.

## Related

<None yet.>

## Sources

- [Legacy wiki.js: Linux hard disk benchmarking tools](../../../../../sources/administration/linux/2026-09-28-storage-and-dns-tools.md) — private source
