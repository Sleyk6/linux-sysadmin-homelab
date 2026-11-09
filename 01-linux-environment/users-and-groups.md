# Users and Groups

## Objective
Create a user and a supplementary group, then verify them using the system's own account databases. Also explain user IDs (UIDs), group IDs (GIDs), home directory permissions, and password status.

## Environment
- **Target:** `telemetry` LXC container (see [System identification](system-identification.md))
- **Run as:** `root` via `pct exec`
- **Captured:** 2026-11-09

## Commands

```bash
groupadd engineers
useradd -m -s /bin/bash -c PitCrew -G engineers pitcrew
```

| Option | Meaning |
|---|---|
| `-m` | Create the home directory and copy in the starter files from `/etc/skel` |
| `-s /bin/bash` | Set the login shell |
| `-c PitCrew` | Set the comment (GECOS) field |
| `-G engineers` | Add the user to a supplementary group |

Both commands printed nothing, which means they succeeded. Like most Unix tools, `useradd` stays quiet when it works.

## Verification and Results

### Identity
```text
$ id pitcrew
uid=1000(pitcrew) gid=1001(pitcrew) groups=1001(pitcrew),1000(engineers)
```
- **Primary group:** Debian created a private group called `pitcrew` automatically (the user private group scheme).
- **Supplementary group:** `engineers` was added by `-G`.

### Account databases
```text
$ getent passwd pitcrew
pitcrew:x:1000:1001:PitCrew:/home/pitcrew:/bin/bash

$ getent group engineers
engineers:x:1000:pitcrew
```

`/etc/passwd` has seven fields, separated by colons:

| Field | Value | Meaning |
|---|---|---|
| 1 | `pitcrew` | Username |
| 2 | `x` | The password hash is stored in `/etc/shadow`, not here |
| 3 | `1000` | UID |
| 4 | `1001` | Primary GID |
| 5 | `PitCrew` | Comment (GECOS) field |
| 6 | `/home/pitcrew` | Home directory |
| 7 | `/bin/bash` | Login shell |

`/etc/group` has four fields: group name, password placeholder, GID, and member list.

### Why the UID and GID don't match
The user's UID is `1000`, but their private group's GID is `1001`. That is because `engineers` was created first and took GID `1000`, the first free regular GID. When `useradd` then tried to create a private group with the same number as the UID, `1000` was taken, so it used the next free GID.

**Lesson:** UIDs and GIDs are numbered independently. Never assume they match. Always check with `id`. This matters when restoring files or sharing storage between systems, because ownership is stored as numbers, not names.

### Home directory
```text
$ ls -ld /home/pitcrew
drwx------ 2 pitcrew pitcrew 4096 Nov  9 18:33 /home/pitcrew

$ ls -A /home/pitcrew
.bash_logout  .bashrc  .profile
```
- **Permissions:** `drwx------` (mode `700`) means only the owner can enter or list the directory. Debian 13 sets this through `HOME_MODE` in `/etc/login.defs`. That is more private than older releases, which used `755`.
- **Starter files:** the three dotfiles were copied from `/etc/skel` by `-m`.

### Password status
```text
$ passwd -S pitcrew
pitcrew L 2026-11-09 0 99999 7 -1
```

| Field | Value | Meaning |
|---|---|---|
| Status | `L` | **Locked.** No password has been set, so password login is impossible. |
| Last change | `2026-11-09` | Date the account was created |
| Min age | `0` | Days before the password may be changed |
| Max age | `99999` | Days before the password must be changed (effectively never) |
| Warn | `7` | Days of warning before the password expires |
| Inactive | `-1` | Grace period after expiry (disabled) |

`chage -l pitcrew` shows the same policy in readable form: the password and account never expire.

**Security note:** a new account that stays locked until an administrator sets a password or installs an SSH key is a safe default. On a production system, password aging (`chage -M 90 -W 14`) and account expiry for temporary staff (`chage -E <date>`) are common hardening steps. These map to the account management objectives in CompTIA Linux+ and Security+.

### Where the user sits among the accounts
```text
$ getent passwd | cut -d: -f1,3 | sort -t: -k2 -n | tail -3
systemd-network:998
pitcrew:1000
nobody:65534
```
On Debian, UIDs below 1000 are system and service accounts (for example `systemd-network`). Regular users start at `1000`. `nobody` (`65534`) is the special unprivileged account. These ranges are set by `UID_MIN` and `UID_MAX` in `/etc/login.defs`.

## Rollback
```bash
userdel -r pitcrew   # also removes the home directory and mail spool
groupdel engineers
```

## Skills Demonstrated
- Creating users and groups with `useradd` and `groupadd`
- Reading `/etc/passwd` and `/etc/group` through `getent`
- Explaining UID and GID allocation and private groups
- Interpreting home directory permissions and the `/etc/skel` starter files
- Auditing password and account aging with `passwd -S` and `chage`
