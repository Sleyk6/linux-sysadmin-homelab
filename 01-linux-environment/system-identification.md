# System Identification

## Objective
Identify the Linux system, user, hostname, kernel, distribution, and shell using basic command-line tools. Also identify how the system is virtualized and which init system is running.

## Environment
- **Target:** `telemetry`, a Debian LXC container (CT 300) on the Proxmox VE server from [Phase 2](../02-proxmox-homelab-server/README.md)
- **Access:** run from a Windows workstation over SSH. Nothing was modified; every command only reads information.

```powershell
# From the Windows workstation (PowerShell)
ssh -t <proxmox-host> "sudo pct exec 300 -- bash -c '<commands>'"
```

## Commands and Results
Captured on 2026-11-09. The local DNS suffix is replaced with `<lan-domain>`.

| Question | Command | Result |
|---|---|---|
| Who am I? | `whoami` | `root` |
| What are my user ID and groups? | `id` | `uid=0(root) gid=0(root) groups=0(root)` |
| What is the hostname? | `hostname` | `telemetry` |
| What is the fully qualified name? | `hostname -f` | `telemetry.<lan-domain>` |
| Which kernel? | `uname -r` | `7.0.14-28-pve` |
| Which CPU architecture? | `uname -m` | `x86_64` |
| Which distribution? | `head -n 5 /etc/os-release` | `Debian GNU/Linux 13 (trixie)`, `VERSION_ID="13"` |
| Which login shell? | `getent passwd root \| cut -d: -f7` | `/bin/bash` |
| Which Bash version? | `bash --version \| head -1` | `GNU bash, version 5.2.37(1)-release` |
| Is this a VM, container, or bare metal? | `systemd-detect-virt` | `lxc` |
| What is PID 1? | `ps -p 1 -o comm=` | `systemd` |
| What is the default boot target? | `systemctl get-default` | `graphical.target` |
| How many CPUs? | `nproc` | `1` |
| How much memory? | `free -h` | `1.0Gi` total, `924Mi` free |
| How much disk? | `df -h /` | `7.8G` total, `757M` used (11%) |
| How long has it been running? | `uptime -p` | `up 1 hour, 34 minutes` |

## What the Results Show

**The kernel belongs to the host.** The `-pve` suffix shows that this is the Proxmox host's kernel. LXC containers do not boot their own kernel. They share the host's kernel and isolate processes using namespaces and cgroups. A virtual machine would report its own distribution kernel instead.

**`systemd-detect-virt` confirms the container type.** It returns `lxc` here, `kvm` inside a KVM virtual machine, and `none` on bare metal. It is the quickest way to answer "where am I running?" on an unfamiliar system.

**`nproc`, `free`, and `df` show the container's limits, not the host's hardware.** The container was created with 1 vCPU, 1 GiB RAM, and an 8 GB root disk. Proxmox enforces those limits with cgroups, and the container's tools report them as if they were the whole machine. The root filesystem is a logical volume on the host's LVM storage (`/dev/mapper/...-vm--300--disk--0`).

**The default target is `graphical.target`, but no desktop is installed.** This is the Debian template's default. With no display manager present, the system simply reaches the same services as `multi-user.target`. This will be revisited in the systemd lab.

**This session ran as `root` (UID 0).** `pct exec` enters a container as root. Day-to-day administration should use an unprivileged account with `sudo`. That is covered in the users-and-groups lab.

## Troubleshooting Note
The first attempt failed with `PRETTY_NAME : The term 'PRETTY_NAME' is not recognized`. The whole Linux command had been wrapped in a PowerShell double-quoted string, and PowerShell does not treat `\"` as an escaped quote, so it ended the string early and tried to run the remainder itself. Removing the inner double quotes and the `$` variables fixed it: `grep -E "..."` became `head -n 5`, and `echo $SHELL` became `getent passwd root | cut -d: -f7`.

**Lesson:** when you nest commands across shells, check which shell parses each layer of quotes first. If quoting gets complicated, open an interactive session instead.
