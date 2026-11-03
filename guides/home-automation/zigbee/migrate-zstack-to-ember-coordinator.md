---
type: guide
title: Migrating a Zigbee2MQTT network from a Z-Stack to an EmberZNet coordinator
description: How to move a Zigbee2MQTT network from a TI Z-Stack coordinator to a Silicon Labs EmberZNet coordinator (e.g. SMLIGHT SLZB) without re-pairing devices.
tags: [zigbee, zigbee2mqtt, slzb, migration, zstack, ember, coordinator]
status: draft
resource:
created: 2026-11-03T20:09:11Z
updated: 2026-11-03T21:15:00Z
generated:
  by: claude/sonnet-5.5
  at: 2026-11-03T21:15:00Z
verified: []
stale_after: 2027-11-03T20:09:11Z
sources:
- id: slzb-coordinator-migration
  resource: 'private:/sources/home-automation/zigbee/2026-11-03-slzb-coordinator-migration.md'
- id: z2m-migration-discussion
  resource: https://github.com/Koenkk/zigbee2mqtt/discussions/31908
- id: z2m-different-chip-feedback
  resource: https://github.com/Koenkk/zigbee2mqtt/discussions/26716
relations: []
superseded_by:
---

# Migrating a Zigbee2MQTT network from a Z-Stack to an EmberZNet coordinator

Move an existing Zigbee network from a Texas Instruments (Z-Stack, `zstack` adapter) coordinator to a
Silicon Labs (EmberZNet, `ember` adapter) coordinator **in place**, keeping the paired devices. Use it
when the old coordinator is failing and the replacement uses the other chip family.

> [!warning] Not officially supported
> Zigbee2MQTT supports backup/restore only within the same chip family. Crossing between Z-Stack
> and EmberZNet works for many networks but community reports are mixed: some devices (often battery
> devices, older Tuya/Aqara) may need a power cycle or re-pairing. Keep every original file.

## Prerequisites

- The old network's `coordinator_backup.json` (Z-Stack format) and `database.db`, plus a copy of
  the complete Zigbee2MQTT data folder. The old coordinator does **not** need to still work, but
  the backup's network frame counter is only as fresh as the file.
- A new Ember-based coordinator (chip family EFR32), reachable on a static IP, with a coordinator
  firmware that exposes the Zigbee serial port over TCP (port `<port>`).
- The old coordinator is permanently switched off. Two coordinators with the same PAN ID and key
  on the same channel conflict.
- `python3` and `openssl` on the machine used for the conversion.

## Steps

### 1. Prepare the new coordinator

In the coordinator's web UI:

- Set a static IP and note the Zigbee TCP port.
- Turn off hardware flow control, **Zigbee socket packet processing** and **Multi-Radio Queue
  Control** where available, and keep **multi-threaded socket** off. If any of them is on, the TCP
  socket connects but the Ember handshake gets no answer.
- Verify the Ember stack answers: start Zigbee2MQTT (step 4) and look for
  `ASH connected` and `EZSP started` in the log.

### 2. Give the new chip the old coordinator's IEEE

Devices cache the old coordinator's IEEE address and reject a coordinator with a different one.
Zigbee-herdsman's attempt to write a custom IEEE to the chip can fail silently. Use the
coordinator vendor's own function (for SLZB devices: "Flash custom IEEE address" in the web UI):

1. Enter the old IEEE exactly as the UI shows the current value, e.g. `0x<16 hex digits>`. A
   colon-separated form may be rejected as "wrong format".
2. Save, then read the value back to confirm it stuck. Some chips allow only one write.

Zigbee2MQTT lists the coordinator IEEE in its database; the byte-reversed form seen in an
ember-written `coordinator_backup.json` is just that file's storage format.

### 3. Convert the backup and put the network parameters in the config

An Ember adapter rejects a Z-Stack backup with `[BACKUP] Current backup file is not for EmberZNet
stack`. Convert it with the [community converter](https://gitlab.com/vwidor/z2m-zstack-to-ember-migration)
(`convert_backup_zstack_to_ember.py`). It keeps the network parameters and swaps the stack-specific
section, using the AES-MMO hash of the default Zigbee 3.0 trust-center link key:

```bash
python3 -I convert_backup_zstack_to_ember.py coordinator_backup.json coordinator_backup.ember.json
# hashed_tclk for the default key "ZigBeeAlliance09": a7a76fa3b83b21641dd3216d6f9ce302
```

Read the script before running it. If the network used a custom trust-center link key, pass
`--tclk-hex <32 hex chars>`.

Then add the **real** network parameters to `configuration.yaml`. If they are missing, Zigbee2MQTT
compares its defaults with the backup, logs `Config does not match backup`, forms a **new** network
and empties the device database:

```yaml
advanced:
  pan_id: <decimal pan id>                # backup pan_id is hex; convert to decimal
  ext_pan_id: [<8 bytes>]                 # backup extended_pan_id with the byte order reversed
  network_key: [<16 bytes>]               # from the backup's network_key
  channel: <channel>
```

Do not paste these values into a wiki, issue tracker or chat; they are the network's security key.

### 4. Switch Zigbee2MQTT over

With the Zigbee2MQTT container **stopped**:

1. Copy the converted file over `coordinator_backup.json`.
2. Make sure `database.db` is the original one with all devices.
3. Edit `configuration.yaml`:

   ```yaml
   serial:
     port: tcp://<coordinator-ip>:<port>
     adapter: ember
   ```

4. Start the container and read the log.

Do all file operations as one chain that stops on the first failure (`set -e`, or `&&` between
every step). Starting the container with only some of the files restored forms a new network.

## Verify

A device list in Zigbee2MQTT proves nothing; check that devices actually transmit:

- Log: `Adapter network matches config` (or a backup restore), then `Currently <n> devices are
  joined`, with `<n>` equal to the old device count. `Forming from config` or `New network
  formed` means the switch failed; stop and restore the original files.
- Within a few minutes mains-powered devices (routers, plugs) should publish. Battery devices
  report when they next wake; pressing a button forces an immediate event.
- A router that stays silent for long, or battery devices that never come back, usually point to
  the frame counter or the IEEE.

## Troubleshooting

### `Failed to start EZSP layer` / the socket connects but the stack never answers

Check the three socket options in step 1 and the coordinator firmware mode. Probe the socket for
an ASH reset reply before involving Zigbee2MQTT.

### `Config does not match backup. Forming from config.`

The `pan_id`, `ext_pan_id` or `network_key` in `configuration.yaml` differ from the backup (or are
absent). A new network was formed and the database emptied: stop the container, restore the
original `database.db`, fix the three values (mind the reversed `ext_pan_id` byte order), start
again.

### Devices appear in Zigbee2MQTT but never report

Expect this when the coordinator IEEE differs from the old one (step 2) or when the coordinator's
outgoing frame counter restarted lower than the value devices remember. Power-cycling a device
makes it rejoin; as a last resort re-pair it.

### Some newer devices cannot be paired afterwards

Devices that request an application link key (for example some Philips Hue dimmers) can fail with
`TC_REQUEST_KEY_TYPE_NOT_SUPPORTED` on EmberZNet coordinators. This is independent of the
migration; existing paired devices keep working.

## Related

- [Setting up SMLIGHT SLZB Zigbee coordinators with Zigbee2MQTT over TCP](slzb-zigbee2mqtt-tcp-setup.md) — the underlying setup, adapter selection and socket settings.
- [Detecting Zigbee2MQTT coordinator hangs](../../../concepts/home-automation/zigbee/zigbee2mqtt-coordinator-hang-detection.md) — why a watchdog that resets the coordinator also shows up in the log as a disconnect.

## Sources

- [Zigbee2MQTT discussion: transparent zstack to ember migration](https://github.com/Koenkk/zigbee2mqtt/discussions/31908)
- [Zigbee2MQTT discussion: feedback on migrating between different chips](https://github.com/Koenkk/zigbee2mqtt/discussions/26716)
- [Private source: SLZB coordinator migration session](../../../../../sources/home-automation/zigbee/2026-11-03-slzb-coordinator-migration.md) — private source (real deployment)
