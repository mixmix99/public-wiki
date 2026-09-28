---
type: guide
title: Pushing backups from TrueNAS to a QNAP NAS with rsync over SSH
description: Set up a key-authenticated rsync push transfer task from TrueNAS to a QNAP NAS, including the QNAP-side SSH and user setup.
tags: [truenas, qnap, rsync, ssh, backup, nas]
status: draft
resource:
created: 2026-09-28T18:54:00Z
updated: 2026-09-28T18:54:00Z
generated:
  by: claude/sonnet-5
  at: 2026-09-28T18:54:00Z
verified: []
stale_after: 2027-09-28T18:54:00Z
sources:
- id: 2026-09-28-rsync-truenas-to-qnap
  resource: 'private:/sources/administration/backup/2026-09-28-rsync-truenas-to-qnap.md'
relations: []
superseded_by:
---

# Pushing backups from TrueNAS to a QNAP NAS with rsync over SSH

Push a one-way copy of data from a TrueNAS system to a QNAP NAS using rsync over SSH with
key-based authentication, scheduled as a native TrueNAS rsync transfer task. This is a data-sync
mechanism, not a full backup solution on its own — it copies data from A to B but has no
versioning or retention of its own.

## Prerequisites

- A TrueNAS system (Core or Scale) with shell/SSH access.
- A QNAP NAS with SSH enabled.
- A dataset/share on each side: a source on TrueNAS, a destination share on the QNAP.

## Steps

### 1. Create a destination share on the QNAP

In the QNAP **Control Panel → Privilege → Shared Folders**, create a shared folder for the backup
data (e.g. `TNbackup`) and restrict its permissions — deny or read-only access for every user
except the dedicated rsync user, so nothing else can accidentally modify or delete the backup
copy.

### 2. Create a dedicated user on the QNAP

Create a user (e.g. `rsync`) and add it to the **administrator** group — required for SSH access
on QNAP — then grant it read/write permissions on the backup share only.

In **Control Panel → Users → Advanced Settings**, enable home folders for all users. In
**Control Panel → Network & File Services**, enable **SSH** and **SFTP**, and set access
permissions there if needed.

Verify SSH login works before continuing:

```bash
ssh rsync@<qnap-ip>
```

### 3. Enable public-key authentication on the QNAP

QNAP ships with public-key auth commented out by default. Edit the sshd config (only `vi` is
available on stock QNAP — no `nano`):

```bash
sudo vi /etc/ssh/sshd_config
```

Uncomment these two lines:

```
PubkeyAuthentication yes
AuthorizedKeysFile  .ssh/authorized_keys
```

Save and exit (`:w`, then `:q`), then prepare the key directory in the rsync user's home folder:

```bash
cd /share/homes/rsync/
mkdir .ssh
chmod 024 .ssh
touch .ssh/authorized_keys
sudo chmod -R 025 .ssh
sudo chown -R rsync .ssh
```

Restart the SSH/rsync services from the QNAP GUI if the settings don't take effect immediately.

### 4. Create a matching user and SSH key on TrueNAS

TrueNAS doesn't create a home directory by default — create one under a dataset first:

```bash
cd /mnt/<pool>/
mkdir -p home/rsync
```

Then create a TrueNAS user (Accounts → Users → Add) with:

```
Username: rsync
Primary group: rsync
Auxiliary groups: <a group with access to the share(s) to be backed up>
Home directory: /mnt/<pool>/home/rsync
```

Generate an SSH key pair as that user (accept all defaults):

```bash
ssh rsync@<truenas-ip>
ssh-keygen
cat .ssh/id_rsa.pub
```

Copy the full public key line, then paste it into the TrueNAS user's **SSH Public Key** field via
the GUI (this also makes it easy to re-add if the key is lost).

### 5. Authorize the key on the QNAP

Log into the QNAP as the `rsync` user and paste the same public key into
`~/.ssh/authorized_keys`:

```bash
ssh rsync@<qnap-ip>
vi .ssh/authorized_keys
```

Test the connection from TrueNAS — it should log in without a password prompt. If it still asks
for one, or the connection fails outright, check that the fingerprint TrueNAS recorded in its
`known_hosts` matches the QNAP's actual host key.

### 6. Create the rsync transfer task on TrueNAS

In the TrueNAS GUI: **Tasks → Rsync Tasks**, create a new task:

| Setting | Value |
|---|---|
| Source path | `/mnt/<pool>/<share-to-back-up>` |
| User | `rsync` |
| Direction | **Push** |
| Remote host | `<qnap-ip>` |
| Rsync mode | SSH |
| Remote SSH port | as configured on the QNAP |
| Remote path | `/share/<destination-share>` |

Disable the compression option — it was found necessary to ensure every file copies correctly.
Leave the delete option off unless a true mirror (rather than an accumulating copy) is desired.

## Verify

Run the task once manually from the TrueNAS UI and confirm the files appear on the QNAP share. If
the task was just created, start a fresh TrueNAS UI/shell session before the first run to avoid
stale-session errors.

## Troubleshooting

- **Passwordless SSH still prompts for a password**: re-check `PubkeyAuthentication`/
  `AuthorizedKeysFile` are uncommented in `sshd_config` and that `.ssh` and `authorized_keys` have
  the exact permissions above — QNAP's SSH daemon is strict about directory/file permissions on
  key files.
- **rsync reports corrupted or incomplete files**: disable the compression option on the transfer
  task.
- **Host key mismatch**: confirm the fingerprint in TrueNAS's `known_hosts` matches the QNAP; a
  QNAP reset or reinstall changes its host key.

## Related

<None yet.>

## Sources

- [Legacy wiki.js (de, translated): rsync from TrueNAS to QNAP](../../../../../sources/administration/backup/2026-09-28-rsync-truenas-to-qnap.md) — private source; German-only wiki.js page, no English original existed
- [Original tutorial this page adapts (homeserver.lu, TrueNAS → Synology)](https://www.homeserver.lu/?p=5807)
