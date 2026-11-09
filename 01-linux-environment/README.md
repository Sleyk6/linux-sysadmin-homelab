# Phase 1 – Building and Configuring a Linux Environment

## Objective
Build and document a functional Linux environment while practicing core Linux system administration skills.

## Skills Covered
- Linux installation and distribution identification
- Command-line navigation
- Files and directories
- Users and groups
- File permissions
- Boot process and GRUB
- systemd services and targets
- Networking
- Linux GUI environments
- Localization, date, time, and NTP
- Troubleshooting

## Environment
| | |
|---|---|
| Operating System | Linux (GNU/Linux) |
| Distribution | Debian GNU/Linux 13 (trixie) |
| Kernel | 7.0.14-28-pve (shared with the Proxmox host) |
| Shell | GNU Bash 5.2.37 |
| Init system | systemd |
| Desktop Environment | None (headless server) |
| Virtualization Platform | Proxmox VE: LXC container `telemetry` (1 vCPU, 1 GiB RAM, 8 GB disk) |

## Labs
| # | Lab | Status |
|---|---|---|
| 1 | [System identification](system-identification.md) | ✅ Complete |
| 2 | [Users and groups](users-and-groups.md) | ✅ Complete |
| 3 | [File permissions](file-permissions.md) | ✅ Complete |

## Progress
- [x] Document system information
- [ ] Configure hostname
- [x] Create users and groups
- [x] Practice file permissions
- [ ] Inspect boot configuration
- [ ] Manage systemd services
- [ ] Verify networking
- [ ] Document GUI environment
- [ ] Configure locale and time
- [ ] Complete troubleshooting notes
