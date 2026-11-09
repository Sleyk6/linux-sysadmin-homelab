# Build steps (with verification and rollback)

Addresses, usernames and key fingerprints are removed. `<server>`, `<desktop-ip>` and `<admin>` are placeholders.

## 1. Prepare
- Backed up the laptop's personal files to the desktop over `scp`, then verified file count and size on both machines.
- Verified the Debian 13 netinst ISO's **SHA256** against the published sum, and wrote it to USB (Rufus, DD mode).

## 2. Install Debian 13 (minimal)
- Guided LVM partitioning; installed **SSH server + standard utilities only**, with no desktop.
- Set the hostname and FQDN in `/etc/hosts` (backup taken first).
```bash
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
```

## 3. Install Proxmox VE on top of Debian
```bash
# add the no-subscription repo (deb822) + Proxmox archive keyring, verify the keyring SHA256
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y proxmox-default-kernel && sudo reboot
sudo apt install -y proxmox-ve postfix open-iscsi chrony   # postfix: local only
sudo apt remove -y linux-image-amd64 'linux-image-6.*' os-prober && sudo update-grub
```
Verify: `uname -r` shows a `-pve` kernel; the web UI on `:8006` loads.

## 4. Storage
```bash
sudo lvcreate -l 95%FREE --thinpool data <vg>
sudo pvesm add lvmthin local-lvm --vgname <vg> --thinpool data --content rootdir,images
```

## 5. Private lab network (NAT out of any uplink)
`/etc/network/interfaces` (backup first):
```
auto vmbr0
iface vmbr0 inet static
        address 10.10.10.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up   iptables -t nat -A POSTROUTING -s 10.10.10.0/24 ! -o vmbr0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s 10.10.10.0/24 ! -o vmbr0 -j MASQUERADE
```
`! -o vmbr0` = translate anything leaving the lab, whichever uplink (Wi-Fi or Ethernet) carries it. The first version used `-o <wifi-if>` and broke when Ethernet became the default route.

## 6. Accounts, MFA, firewall
- `sudo` for the admin user; a personal Proxmox admin (`pveum`) with **TOTP**. Recovery codes stored offline.
- Datacenter firewall **on**: IPSet `management` = `<desktop-ip>` only, with inbound 22/8006 allowed from that set.

## 7. SSH keys only
```bash
# desktop: ed25519 key with passphrase, loaded in ssh-agent; ~/.ssh/config Host entry
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t && sudo systemctl reload ssh
```
Verify: a new key login works, and `ssh -o PubkeyAuthentication=no ...` returns `Permission denied (publickey)`.
Rollback: delete the file and `systemctl reload ssh`.

## 8. Lab container
```bash
sudo pveam download local debian-13-standard_13.6-1_amd64.tar.zst
sudo pct create 300 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst --hostname telemetry \
  --unprivileged 1 --features nesting=1 --cores 1 --memory 1024 --swap 512 --rootfs local-lvm:8 \
  --net0 name=eth0,bridge=vmbr0,ip=10.10.10.30/24,gw=10.10.10.1 --start 1
```
GitHub: a deploy key generated *inside* the container (the private key never leaves it), GitHub's host key fingerprint checked, and `.gitignore` added before the first commit.

## 9. Run headless
```bash
printf '[Login]\nHandleLidSwitch=ignore\nHandleLidSwitchExternalPower=ignore\nHandleLidSwitchDocked=ignore\n' | sudo tee /etc/systemd/logind.conf.d/lid.conf
sudo systemctl restart systemd-logind
```

## 10. Backup, restore test, off-host copy
```bash
sudo vzdump 300 --storage local --mode snapshot --compress zstd
sudo pvesh create /cluster/backup --schedule "02:30" --vmid 300 --storage local --mode snapshot \
  --compress zstd --prune-backups keep-last=7 --enabled 1
sudo pct restore 399 /var/lib/vz/dump/<archive>.tar.zst --storage local-lvm --hostname restore-test --start 0
sudo pct destroy 399 --purge
```
Desktop (PowerShell): `scp` the newest archive, then compare `sha256sum` on the server with `Get-FileHash` on Windows. They match.

## 11. Wired uplink (USB gigabit Ethernet)
- Identified the adapter (`ip -br link`, `ethtool`, driver `cdc_ncm`), confirmed link at 1000 Mb/s.
- Read the router's DHCP range first, then chose a static address in a reserved range **outside** it, after an ARP check that it was free.

`/etc/network/interfaces` (backup first; no `auto` line, the unit below brings it up):
```
iface vmbr1 inet static
        address <lan-ip>/24
        gateway <router-ip>
        bridge-ports <usb-nic>
        bridge-stp off
        bridge-fd 0
```
`/etc/systemd/system/paddock-usb-eth.service`:
```
[Unit]
Description=Bring up vmbr1 when the USB Ethernet adapter appears
BindsTo=sys-subsystem-net-devices-<usb-nic>.device
After=sys-subsystem-net-devices-<usb-nic>.device networking.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/ifup vmbr1

[Install]
WantedBy=sys-subsystem-net-devices-<usb-nic>.device
```
```bash
sudo systemctl daemon-reload && sudo systemctl enable --now paddock-usb-eth
```
Verify: `systemctl status paddock-usb-eth` = active (exited), status=0/SUCCESS; `ip route` default via `vmbr1`; a full **reboot test** brings everything back with no manual steps. Desktop `~/.ssh/config` points `paddock` at the wired address and keeps a `paddock-wifi` alias as a fallback.
Rollback: `sudo systemctl disable --now paddock-usb-eth`, restore the interfaces backup.

> Never edit host networking in the Proxmox web UI *and* by hand: the UI saves `/etc/network/interfaces.new`, which replaces the real file at the next boot.
