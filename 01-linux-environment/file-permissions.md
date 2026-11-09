# File Permissions

## Objective
Build a shared team directory that only members of one group can use, then prove the access rules by testing as a member and as an outsider. Covers `chmod`, `chgrp`, octal and symbolic modes, the setgid bit, and `umask`.

## Environment
- **Target:** `telemetry` LXC container
- **Accounts:** `pitcrew`, a member of `engineers` (from [Users and groups](users-and-groups.md)); `nobody`, the built-in unprivileged account, used as the outsider
- **Captured:** 2026-11-09

## Commands

```bash
mkdir -p /srv/garage
chgrp engineers /srv/garage
chmod 2770 /srv/garage
runuser -u pitcrew -- sh -c umask
runuser -u pitcrew -- touch /srv/garage/setup-sheet.txt
runuser -u nobody -- ls /srv/garage
runuser -u nobody -- ls /home/pitcrew
chmod 640 /srv/garage/setup-sheet.txt
stat -c %A_%a_%U:%G_%n /srv/garage/setup-sheet.txt
```

`runuser -u <user> -- <command>` lets root run a single command as another user, without a password. It is a clean way to test permissions from each user's point of view.

## Results

### 1. Shared directory
```text
drwxrws--- 2 root engineers 4096 Nov  9 18:34 /srv/garage
```

Mode `2770` breaks down like this:

| Digit | Applies to | Value | Effect |
|---|---|---|---|
| `2` | special bit | setgid | New files inherit the directory's group (`engineers`) |
| `7` | owner (`root`) | `rwx` | Full access |
| `7` | group (`engineers`) | `rwx` | Full access for team members |
| `0` | others | `---` | No access at all |

The setgid bit appears as `s` in the group execute position (`rws`). Each permission is a number: read = 4, write = 2, execute = 1. Add them up per class, so `rwx` is 7 and `r--` is 4.

### 2. Default permissions (umask)
```text
$ umask      # as pitcrew
0022
```
New files start from `666` and new directories from `777`. The umask removes bits from that starting point: `666 - 022 = 644`, so new files come out as `rw-r--r--`.

### 3. A team member creates a file
```text
-rw-r--r-- 1 pitcrew engineers 0 Nov  9 18:34 setup-sheet.txt
```
- **Mode `644`:** exactly what the umask predicted.
- **Group `engineers`, not `pitcrew`:** `pitcrew`'s primary group is `pitcrew`, so without setgid the file would have belonged to that group. The setgid bit on the directory made it inherit `engineers`. This is what keeps team folders shareable.

### 4. An outsider is refused
```text
$ ls /srv/garage      # as nobody
ls: cannot open directory '/srv/garage': Permission denied

$ ls /home/pitcrew    # as nobody
ls: cannot open directory '/home/pitcrew': Permission denied
```
Both refusals are the intended result:
- `/srv/garage` gives others `---`.
- `/home/pitcrew` is `700` (see Lab 2).

The outsider cannot even list the file names, let alone read the contents.

### 5. Tightening a file
```text
-rw-r-----_640_pitcrew:engineers_/srv/garage/setup-sheet.txt
```
Mode `640` gives:
- **Owner:** read and write
- **Team:** read only
- **Others:** nothing

`stat` shows the same mode in both forms, symbolic (`-rw-r-----`) and octal (`640`), which makes it a good check.

## Key Takeaways
- **Directory permissions control access to the names inside.** Without `x` (search), you cannot enter a directory or reach any file in it, whatever the file's own mode.
- **Least privilege:** others get `0` by default on shared data. Access is granted to a group, never to the world.
- **setgid on shared directories** prevents files from ending up owned by each user's private group.
- **Verify as the affected user** with `runuser` or `sudo -u`. Do not just read the mode bits and assume.

## Rollback
```bash
rm -r /srv/garage
```

## Skills Demonstrated
`chmod` (octal and special bits), `chgrp`, setgid, `umask`, `stat`, testing permissions per user with `runuser`, least-privilege design. These map to the file permissions and access control objectives of CompTIA Linux+ and Security+.
