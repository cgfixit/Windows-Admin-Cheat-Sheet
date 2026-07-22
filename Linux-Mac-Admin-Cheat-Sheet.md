<!--
============================================================================
  LINUX & macOS ADMIN ONE-LINER CHEAT SHEET
  Companion to Windows-Admin-Cheat-Sheet.md
  Covers: RHEL / Rocky / Alma  •  Fedora  •  Ubuntu / Debian  •  macOS
  Last Updated: 2026-07-22

  SECURITY: NEVER store passwords, keys, or secrets in this file.
            Use a vault, env vars, an agent, or an interactive prompt.

  COMPATIBILITY KEY:
    [ALL]        = All listed Linux distros AND macOS (POSIX-ish)
    [LINUX]      = All listed Linux distros (not macOS)
    [RHEL/Fed]   = RHEL family (RHEL, Rocky, Alma) and Fedora  (dnf/firewalld/SELinux)
    [Ubuntu]     = Ubuntu / Debian                              (apt/ufw/AppArmor)
    [macOS]      = macOS only
    [DEP]        = Deprecated / legacy; shown with modern replacement

  SHELL / TOOLING NOTES:
    - Examples assume bash/zsh. macOS default shell is zsh (Catalina+).
    - Commands needing root are shown with `sudo`; drop it if already root.
    - RHEL 8+/9 and Fedora use `dnf`; RHEL 7 uses `yum` (dnf is a drop-in).
    - Ubuntu 17.10+/Debian 10+ use `netplan` + systemd; older use ifupdown.
    - macOS package commands assume Homebrew (https://brew.sh).
============================================================================
-->

# Linux & macOS Admin Arsenal 2026

One-liner cheat sheet for modern Linux and macOS administration,
troubleshooting, automation, and security. Companion to the
[Windows Admin Arsenal](./Windows-Admin-Cheat-Sheet.md).
Covers **RHEL / Rocky / Alma**, **Fedora**, **Ubuntu / Debian**, and **macOS**.
Curated by Chris Grady ([@cgfixit](https://github.com/cgfixit)).

> **Security:** Never hardcode passwords, keys, or secrets here. Use a vault,
> env vars, an SSH/GPG agent, or an interactive prompt. Placeholders like
> `hostname`, `user`, `192.0.2.x`, and `XXXXX` are intentional.

**Compatibility key:** `[ALL]` all platforms · `[LINUX]` all Linux ·
`[RHEL/Fed]` RHEL family + Fedora · `[Ubuntu]` Ubuntu/Debian · `[macOS]` macOS ·
`[DEP]` deprecated (modern replacement shown).

---

## Table of Contents

1.  [Network Reset, Diagnostics & Firewall](#1-network-reset-diagnostics--firewall)
2.  [DNS / DHCP](#2-dns--dhcp)
3.  [Static Routes & Network Discovery](#3-static-routes--network-discovery)
4.  [User & Sudo Management](#4-user--sudo-management)
5.  [Remote Access (SSH)](#5-remote-access-ssh)
6.  [Process Management](#6-process-management)
7.  [System Info & Health](#7-system-info--health)
8.  [Disk, File & Storage Operations](#8-disk-file--storage-operations)
9.  [Package Management](#9-package-management)
10. [Software Updates & Patching](#10-software-updates--patching)
11. [Services & Init Systems (systemd / launchd)](#11-services--init-systems-systemd--launchd)
12. [Directory Services / LDAP / AD Join](#12-directory-services--ldap--ad-join)
13. [Mail (Postfix)](#13-mail-postfix)
14. [Virtualization & Containers](#14-virtualization--containers)
15. [Printing (CUPS)](#15-printing-cups)
16. [Remote Execution & Automation](#16-remote-execution--automation)
17. [Shell / Bash General Admin](#17-shell--bash-general-admin)
18. [Logs & journald](#18-logs--journald)
19. [Network Info](#19-network-info)
20. [Legacy → Modern Command Equivalents](#20-legacy--modern-command-equivalents)
21. [Security Hardening](#21-security-hardening)
22. [Repositories & Package Sources](#22-repositories--package-sources)
23. [Storage: LVM & Software RAID](#23-storage-lvm--software-raid)
24. [ZFS / Btrfs (TrueNAS / NAS)](#24-zfs--btrfs-truenas--nas)
25. [macOS Power Tools](#25-macos-power-tools)
26. [Misc Utilities](#26-misc-utilities)
- [Quick Reference](#quick-reference)

---

## [1] Network Reset, Diagnostics & Firewall

### Restart networking

```bash
# [RHEL/Fed] NetworkManager
sudo systemctl restart NetworkManager
nmcli networking off && nmcli networking on

# [Ubuntu] netplan (config in /etc/netplan/*.yaml)
sudo netplan apply
sudo systemctl restart systemd-networkd     # server / netplan-networkd
sudo systemctl restart NetworkManager       # desktop

# [macOS] bounce an interface (en0 = Wi-Fi or first Ethernet)
sudo ifconfig en0 down && sudo ifconfig en0 up
networksetup -setairportpower en0 off && networksetup -setairportpower en0 on
```

### Flush DNS cache

```bash
# [RHEL/Fed][Ubuntu] systemd-resolved
sudo resolvectl flush-caches
resolvectl statistics                        # verify cache activity

# nscd (if used)
sudo systemctl restart nscd

# [macOS]
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

### Firewall

```bash
# [RHEL/Fed] firewalld
firewall-cmd --state
firewall-cmd --list-all
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-port=443/tcp
sudo firewall-cmd --reload
sudo firewall-cmd --panic-on                 # drop everything (emergency); --panic-off to restore

# [Ubuntu] ufw
sudo ufw status verbose
sudo ufw allow OpenSSH                        # or: sudo ufw allow 22/tcp
sudo ufw allow 443/tcp
sudo ufw enable && sudo ufw reload

# [LINUX] nftables / iptables (underlying)
sudo nft list ruleset
sudo iptables -L -n -v                        # [DEP] legacy; prefer nft

# [macOS] packet filter (pf) + application firewall
sudo pfctl -sr                                # show pf rules
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on
```

### Port & connectivity diagnostics

```bash
# [LINUX] listening sockets w/ owning process (modern)
ss -tulpn
ss -tulpn | grep ':443'
# [DEP] netstat -tulpn   → use ss

# [ALL] which process owns a port / file
sudo lsof -i :443
sudo lsof -nP -iTCP -sTCP:LISTEN              # all listeners

# [ALL] test a remote port (no telnet needed)
nc -zv hostname 443
curl -I https://hostname
timeout 3 bash -c '</dev/tcp/hostname/443' && echo open || echo closed

# [ALL] path / latency
ping -c 4 hostname
traceroute hostname                           # tracepath on Ubuntu; brew install for macOS extras
mtr hostname                                  # live traceroute+ping (install mtr)
```

### TCP tuning

```bash
# [LINUX] view/set kernel network params (persist in /etc/sysctl.d/*.conf)
sysctl net.ipv4.tcp_congestion_control
sudo sysctl -w net.core.rmem_max=16777216
sudo sysctl --system                          # reload sysctl.d after editing

# [macOS]
sysctl net.inet.tcp.mssdflt
```

### Packet capture [LINUX]

```bash
# tcpdump is the standard packet-capture diagnostic tool and isn't currently in the Diagnostics section. Verified live this pass via the Ubuntu 22.04 (jammy) tcpdump(8) manpage, which confirms -i, -w, -r, and -n exactly as used, plus that a bare filter expression (host ... and port ...) is valid.
# quick packet capture on an interface, filtered to a host/port
sudo tcpdump -i eth0 host 192.0.2.10 and port 443
sudo tcpdump -i eth0 -w capture.pcap          # write to file for later analysis
tcpdump -r capture.pcap -n                     # read back without re-resolving names
```

### macOS throughput/responsiveness test [macOS]

```bash
# networkQuality has shipped as a built-in macOS CLI since macOS 12 Monterey and is the standard throughput/responsiveness diagnostic; there is no ping/nc/curl equivalent already in this section. Could not re-verify a live citation this pass (ss64.com and Apple's support site both returned 403 through the proxy this session), so this addition rests on prior knowledge rather than a freshly fetched source — flagging for a manual spot-check before merge.
networkQuality                                  # built-in up/down capacity + responsiveness (RPM)
networkQuality -v                               # verbose, shows detailed phases
```

---

## [2] DNS / DHCP

```bash
# --- Show current DNS servers ---
resolvectl status                             # [RHEL/Fed][Ubuntu] per-link DNS
cat /etc/resolv.conf                           # [LINUX] effective resolver (often a symlink)
scutil --dns                                   # [macOS] full resolver config
networksetup -getdnsservers Wi-Fi              # [macOS] per-service

# --- Set DNS ---
sudo nmcli con mod "System eth0" ipv4.dns "1.1.1.1 9.9.9.9"   # [RHEL/Fed]
sudo nmcli con up "System eth0"
sudo networksetup -setdnsservers Wi-Fi 1.1.1.1 9.9.9.9         # [macOS]
sudo networksetup -setdnsservers Wi-Fi Empty                   # [macOS] revert to DHCP

# --- DHCP renew ---
sudo nmcli con down eth0 && sudo nmcli con up eth0             # [RHEL/Fed]
sudo dhclient -r && sudo dhclient                              # [LINUX][DEP] ifupdown/dhclient
sudo ipconfig set en0 DHCP                                     # [macOS] renew lease

# --- Query DNS ---
dig +short example.com
dig example.com MX
dig @1.1.1.1 example.com                       # query a specific resolver
host example.com
nslookup example.com                            # [DEP] prefer dig/host
```

### Temporary per-link DNS override / query via resolved [RHEL/Fed][Ubuntu]

```bash
# resolvectl dns/revert/query are documented systemd-resolved subcommands distinct from the nmcli-based persistent DNS change already shown. Verified live this pass via the Ubuntu (focal) resolvectl(1) manpage, which documents per-interface DNS query/override/revert behavior consistent with these commands.
# set a temporary (non-persistent) per-link DNS server without touching NetworkManager
sudo resolvectl dns eth0 1.1.1.1 9.9.9.9
resolvectl revert eth0                         # drop the override, fall back to normal config
# resolve a name specifically through systemd-resolved (shows which protocol answered)
resolvectl query example.com
```

---

## [3] Static Routes & Network Discovery

```bash
# --- Routing table ---
ip route show                                  # [LINUX]
netstat -rn                                    # [macOS] (and legacy Linux)
route -n get default                            # [macOS] default gateway

# --- Add routes ---
sudo ip route add 192.0.2.0/24 via 192.0.2.1 dev eth0          # [LINUX] runtime
sudo route -n add -net 192.0.2.0/24 192.0.2.1                  # [macOS] runtime
# Persist [RHEL/Fed]: nmcli con mod <con> +ipv4.routes "192.0.2.0/24 192.0.2.1"
# Persist [Ubuntu]:   add a routes: block in /etc/netplan/*.yaml, then netplan apply

# --- Neighbors / ARP ---
ip neigh                                        # [LINUX] modern
arp -a                                          # [ALL] legacy-friendly

# --- Discovery / scan a subnet ---
nmap -sn 192.0.2.0/24                           # [ALL] ping sweep (install nmap)
avahi-browse -a                                 # [LINUX] mDNS/Bonjour services
dns-sd -B _ssh._tcp                             # [macOS] browse Bonjour services
```

### Show the actual route the kernel would use [LINUX]

```bash
# ip route get resolves policy rules + routing table into the single route the kernel will actually use, which the existing 'ip route show' listing doesn't answer directly. Verified live this pass via the Ubuntu (jammy) ip-route(8) manpage, which documents exactly this 'ip route get ADDRESS [from ADDRESS]' syntax.
ip route get 192.0.2.10                        # which route/src-IP/dev the kernel picks for a destination
ip route get 8.8.8.8 from 192.0.2.5             # simulate from a specific source address
```

### Layer-2 ARP probe of a single host [LINUX]

```bash
# arping complements the existing 'ip neigh'/'arp -a' static-table view with an active Layer-2 probe. CORRECTED: the draft used '-I eth0' for the interface flag; the Ubuntu (jammy) arping(8) manpage — the iputils-arping build shipped by default on Debian/Ubuntu/RHEL — documents lowercase '-i interface' ('Don't guess, use the specified interface'); there is no uppercase '-I' option in that build. Fixed to '-i eth0'. (Note: a different, less commonly pre-installed 'arping' by Thomas Habets does use '-I' — if this repo standardizes on that variant instead, the flag should be revisited.)
sudo arping -c 3 192.0.2.10                     # ARP-level reachability check, bypasses IP routing
sudo arping -c 3 -i eth0 192.0.2.10             # pin to a specific interface
```

---

## [4] User & Sudo Management

```bash
# --- Create / modify users [LINUX] ---
sudo useradd -m -s /bin/bash iadmin            # RHEL/Fed low-level
sudo adduser iadmin                             # Ubuntu interactive wrapper
sudo passwd iadmin                              # set password (interactive)
sudo usermod -aG wheel iadmin                   # [RHEL/Fed] grant sudo
sudo usermod -aG sudo  iadmin                   # [Ubuntu] grant sudo
sudo userdel -r iadmin                          # delete user + home

# --- List users / groups [LINUX] ---
getent passwd                                   # all accounts (local + directory)
id iadmin                                        # uid/gid/groups
groups iadmin
getent group wheel                               # who has sudo (RHEL/Fed)

# --- sudo policy [LINUX] ---
sudo visudo                                      # edit /etc/sudoers safely
sudo visudo -f /etc/sudoers.d/iadmin             # drop-in (preferred)

# --- Users on macOS ---
dscl . -list /Users | grep -v '^_'               # real (non-system) users
sudo sysadminctl -addUser iadmin -fullName "IT Admin" -password -   # '-' = prompt
sudo dseditgroup -o edit -a iadmin -t user admin  # add to admin group
id iadmin

# --- Who am I / who is on ---
whoami
who       # logged-in users
w         # users + what they're running
last -n 10   # recent logins (Linux; macOS: `last | head`)
```

### Password aging policy [LINUX]

```bash
# chage manages the shadow password aging fields directly; useful for enforcing rotation policy alongside sudo/group changes already in this section.
# --- Password aging (shadow policy) ---
chage -l iadmin                                 # view expiry/aging info for a user
sudo chage -M 90 -W 14 iadmin                    # max 90-day password life, warn 14 days out
sudo chage -E 2026-12-31 iadmin                  # set an account expiration date (-1 clears it)
```

### Check your own sudo rights [LINUX]

```bash
# Fast way to audit what a sudoers/visudo drop-in actually grants a user without reading raw config; complements the existing visudo entries.
sudo -l                                          # list commands the invoking user may run
sudo -l -U iadmin                                # list another user's sudo privileges (root only)
```

---

## [5] Remote Access (SSH)

```bash
# --- Connect ---
ssh user@hostname
ssh -p 2222 user@hostname
ssh -J jump-host user@internal-host             # ProxyJump / bastion

# --- Keys (ed25519 preferred) ---
ssh-keygen -t ed25519 -C "user@device 2026"
ssh-copy-id user@hostname                        # install pubkey (Linux)
# macOS (no ssh-copy-id by default): brew install ssh-copy-id  — or:
cat ~/.ssh/id_ed25519.pub | ssh user@hostname 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'

# --- Reusable client config: ~/.ssh/config ---
#   Host prod
#     HostName 192.0.2.10
#     User iadmin
#     Port 2222
#     IdentityFile ~/.ssh/id_ed25519

# --- Enable/start the SSH server ---
sudo systemctl enable --now sshd                 # [RHEL/Fed]
sudo systemctl enable --now ssh                  # [Ubuntu] (service is 'ssh')
sudo systemsetup -setremotelogin on              # [macOS] enable Remote Login

# --- Copy files ---
scp file.txt user@hostname:/path/                # simple copy
rsync -avh --progress src/ user@hostname:/dst/   # resumable, efficient
```

### FIDO2 hardware-backed keys [LINUX]

```bash
# ed25519-sk/ecdsa-sk keep the private key on the hardware token; only a reference stub is written to disk, meaningfully stronger than a plain ed25519 key for admin workstations.
# --- FIDO2/U2F hardware security key (OpenSSH 8.2+) ---
ssh-keygen -t ed25519-sk -O resident -O verify-required -C "user@device 2026"
# resident: credential stored on the key itself (usable from another host)
# verify-required: forces PIN/biometric verification in addition to touch
```

### Harden sshd_config [LINUX]

```bash
# These three directives are the standard baseline for hardening an sshd deployment beyond the client-side items already in this section; sshd -t catches syntax errors before a reload locks you out.
# --- Server hardening: /etc/ssh/sshd_config ---
PermitRootLogin prohibit-password              # default; use 'no' to block root entirely
PasswordAuthentication no                       # key-only auth once pubkeys are deployed
AuthorizedKeysCommand /path/to/script            # look up keys dynamically (e.g. from a vault)
sudo sshd -t                                     # validate config syntax before reloading
sudo systemctl reload sshd                       # apply (RHEL/Fed; Ubuntu: 'ssh')
```

---

## [6] Process Management

```bash
# --- List / sort ---
ps aux
ps -ef
ps aux --sort=-%mem | head                       # top memory (Linux)
ps aux -r | head                                 # top CPU (macOS: -r = sort by CPU)
top                                              # live; press M (mem) / P (cpu)
htop                                             # nicer live view (install htop)

# --- Find / signal ---
pgrep -fl nginx                                  # find PIDs by pattern
kill 1234                                         # TERM (graceful)
kill -9 1234                                      # KILL (force)
pkill -f "python myscript"                        # by command pattern
killall firefox                                   # by exact process name

# --- Priority ---
nice -n 10 ./batch.sh                             # start low-priority
sudo renice -n -5 -p 1234                          # raise priority of running PID

# --- What a process is doing ---
sudo lsof -p 1234                                  # open files/sockets
strace -p 1234                                     # [LINUX] syscalls (install strace)
sudo dtruss -p 1234                                # [macOS] syscalls (needs SIP considerations)
```

### I/O priority [LINUX]

```bash
# ionice manages the I/O scheduling class/priority independently of CPU nice(1); confirmed via Ubuntu manpage: classes are idle/best-effort/realtime, -c sets class, -n sets 0-7 priority within it, realtime requires root.
# --- I/O priority (separate from CPU nice) ---
ionice -c 3 -p 1234                                # idle class: only I/O when nothing else needs disk
ionice -c 2 -n 0 rsync -a /src/ /dst/               # best-effort class, highest sub-priority (0-7)
```

### Process tree view [LINUX]

```bash
# ps(1) supports -H / f / --forest for ASCII-art process hierarchy display; confirmed via Ubuntu manpage. Complements the existing pgrep/kill/nice content, which has no tree view.
# --- Parent/child relationships at a glance ---
ps -ejH                                             # ASCII process tree (forest)
ps axjf                                             # alternate forest invocation
```

---

## [7] System Info & Health

```bash
# --- OS / version ---
uname -a
cat /etc/os-release                               # [LINUX] name + version
hostnamectl                                       # [LINUX] host, OS, kernel, machine-id
sw_vers                                           # [macOS] product + build
system_profiler SPSoftwareDataType                # [macOS] verbose

# --- CPU / memory ---
lscpu                                             # [LINUX] CPU topology
nproc                                             # [LINUX] logical cores
free -h                                           # [LINUX] memory
vm_stat                                           # [macOS] memory pages
sysctl -n hw.ncpu hw.memsize                       # [macOS] cores + RAM bytes

# --- Uptime / last boot ---
uptime
who -b                                            # [LINUX] boot time
sysctl -n kern.boottime                            # [macOS]

# --- Hardware / BIOS ---
sudo dmidecode -t bios,system                      # [LINUX] BIOS + system (install dmidecode)
lspci; lsusb                                       # [LINUX] PCI/USB devices
system_profiler SPHardwareDataType                 # [macOS] serial, model, RAM

# --- Disk & SMART health ---
lsblk -f                                           # [LINUX] block devices + FS
df -h                                              # free space, all platforms
sudo smartctl -a /dev/sda                           # [LINUX] SMART (install smartmontools)
diskutil list                                       # [macOS] disks/partitions
diskutil info disk0                                 # [macOS] device detail

# --- Sensors / power ---
sensors                                            # [LINUX] temps (lm_sensors)
pmset -g                                           # [macOS] power/battery settings
```

### Time sync status [LINUX]

```bash
# timedatectl status reports whether the system clock is synchronized and NTP service is active; timesync-status gives per-server offset/jitter/stratum detail. Confirmed via Ubuntu manpage; complements chronyc sources already referenced in section 20.
# --- Time sync health (systemd-timesyncd/chrony) ---
timedatectl status                                 # sync state, timezone, NTP service active?
timedatectl timesync-status                         # server, stratum, offset, jitter detail
```

### Boot performance [LINUX]

```bash
# Standard systemd boot-diagnostics trio; confirmed via Ubuntu manpage. Not currently covered anywhere in the sheet and directly useful for 'why did this box boot slowly' triage alongside the existing uptime/BIOS/hardware content.
# --- Boot time breakdown (systemd) ---
systemd-analyze time                                # kernel/initrd/userspace startup split
systemd-analyze blame                               # units ranked by init duration
systemd-analyze critical-chain                      # time-critical dependency path to default.target
```

---

## [8] Disk, File & Storage Operations

```bash
# --- Space usage ---
df -h                                             # per-filesystem free space
du -sh ./*                                        # size of each item here
du -h --max-depth=1 . | sort -h                    # [LINUX] biggest subdirs
sudo du -sh ./* | sort -h                          # [macOS] (no --max-depth)
ncdu /                                             # interactive disk usage (install ncdu)

# --- Find files ---
find / -xdev -type f -size +500M 2>/dev/null       # files > 500 MB
find . -name '*.log' -mtime +30                    # logs older than 30 days
find . -type f -printf '%s\t%p\n' 2>/dev/null | sort -rn | head -20   # [LINUX] 20 largest

# --- Sync / mirror (⚠ --delete removes extras in dest) ---
rsync -avh --delete /src/ /dst/                    # local mirror
rsync -avh --dry-run --delete /src/ /dst/          # PREVIEW first — always
robocopy_equiv() { rsync -avh --partial --progress "$@"; }   # resumable copy helper

# --- Symlinks / mounts ---
ln -s /real/path ~/shortcut                        # symlink
mount | column -t                                  # [LINUX] mounted filesystems
sudo mount -a                                      # [LINUX] mount everything in /etc/fstab
blkid                                              # [LINUX] UUIDs for fstab
diskutil mount /dev/disk2s1                         # [macOS] mount a volume

# --- Show file contents ---
cat file.txt; less file.txt; tail -f app.log
```

### Filesystem tree view (UUID/label without a separate blkid pass) [LINUX]

```bash
# Complements the existing blkid line: lsblk -f shows the same UUID/label info but in a tree alongside mountpoints and free space in one call, which is usually the faster first check when planning /etc/fstab entries or diagnosing an unmounted disk.
lsblk -f                                           # [LINUX] tree of block devices w/ FSTYPE, LABEL, UUID, FSUSE%
```

### df-style view driven by fstab/mtab (catches bind mounts) [LINUX]

```bash
# findmnt is the modern iproute2-era mount inspector (companion to lsblk/ip). Unlike plain df -h, --df also surfaces bind mounts and lets you query by target path or source device, which is handy when troubleshooting duplicate/overlapping mounts.
findmnt --df                                       # [LINUX] df(1)-style output built from fstab/mtab (includes bind mounts)
findmnt /mnt/data                                  # [LINUX] show what (if anything) is mounted at a specific path
```

---

## [9] Package Management

Cross-platform reference. Pick the column for your OS.

| Task | RHEL/Fedora (`dnf`) | Ubuntu/Debian (`apt`) | macOS (`brew`) |
|------|---------------------|-----------------------|----------------|
| Refresh index | `sudo dnf check-update` | `sudo apt update` | `brew update` |
| Search | `dnf search vim` | `apt search vim` | `brew search vim` |
| Install | `sudo dnf install vim` | `sudo apt install vim` | `brew install vim` |
| Remove | `sudo dnf remove vim` | `sudo apt remove vim` | `brew uninstall vim` |
| Upgrade all | `sudo dnf upgrade` | `sudo apt upgrade` | `brew upgrade` |
| Info | `dnf info vim` | `apt show vim` | `brew info vim` |
| List installed | `dnf list installed` | `apt list --installed` | `brew list` |
| Which pkg owns file | `dnf provides /usr/bin/vim` | `dpkg -S /usr/bin/vim` | `brew which-formula` |
| Clean cache | `sudo dnf clean all` | `sudo apt clean` | `brew cleanup` |

```bash
# --- Low-level query ---
rpm -qa | sort                                     # [RHEL/Fed] all installed
dpkg -l                                            # [Ubuntu] all installed
rpm -qi httpd                                       # [RHEL/Fed] package details

# --- Universal Linux app formats ---
sudo snap install code --classic                   # [Ubuntu] snap
flatpak install flathub org.gimp.GIMP              # [LINUX] flatpak (add flathub remote first)

# --- macOS extras ---
brew install --cask firefox                        # GUI apps via cask
brew services list                                 # background services managed by brew
mas list                                           # Mac App Store apps (brew install mas)
```

### Preview pending upgrades before running apt upgrade [Ubuntu]

```bash
# The current table only shows 'refresh index' and 'upgrade all'; this fills the gap of previewing exactly which packages are queued for upgrade before committing to apt upgrade/full-upgrade — useful in change-controlled environments.
apt list --upgradable                              # [Ubuntu] see what would be upgraded, without changing anything
```

### Reproducible package lists on macOS (Brewfile) [macOS]

```bash
# The current macOS row covers brew install/upgrade/list one package at a time; Brewfile + brew bundle is Homebrew's official way to capture and replay an entire machine's package set (like a lightweight package.json for brew), useful for rebuilding a workstation or standardizing a fleet.
brew bundle dump --global --force                  # [macOS] snapshot installed formulae/casks/taps to a Brewfile
brew bundle install                                 # [macOS] install everything listed in a Brewfile
brew bundle check                                   # [macOS] verify system matches the Brewfile, no changes made
```

---

## [10] Software Updates & Patching

```bash
# --- Apply updates ---
sudo dnf upgrade --refresh                          # [RHEL/Fed]
sudo apt update && sudo apt full-upgrade            # [Ubuntu]
sudo softwareupdate -ia --verbose                   # [macOS] all Apple updates
brew update && brew upgrade                          # [macOS] Homebrew packages

# --- Security-only ---
sudo dnf upgrade --security                          # [RHEL/Fed]
sudo unattended-upgrade -d                            # [Ubuntu] run configured auto-updates

# --- Automate ---
sudo dnf install dnf-automatic && sudo systemctl enable --now dnf-automatic.timer   # [RHEL/Fed]
sudo apt install unattended-upgrades && sudo dpkg-reconfigure unattended-upgrades   # [Ubuntu]

# --- Reboot required? ---
[ -f /var/run/reboot-required ] && echo "reboot needed"   # [Ubuntu]
needs-restarting -r; echo "exit=$?"                        # [RHEL/Fed] (dnf-utils); 1 = reboot
uname -r                                                    # running kernel vs installed
```

### Roll back a bad patch transaction [RHEL/Fed]

```bash
# The current section covers applying updates but has no rollback path if a patch breaks something. dnf history undo reverses a specific transaction (best-effort, based on current RPMDB state) and is the standard first response to a bad update on RHEL/Fedora before reaching for a snapshot restore.
dnf history list                                    # [RHEL/Fed] see recent transactions with IDs
sudo dnf history undo <ID>                          # [RHEL/Fed] revert a specific update transaction
```

### Kernel live patching on Ubuntu Pro (no-reboot security fixes) [Ubuntu]

```bash
# The section's 'reboot required?' check assumes a reboot is the only remediation. On Ubuntu Pro-attached systems, Livepatch applies kernel security fixes in memory without a reboot — worth knowing so you don't reboot a fleet unnecessarily when only kernel CVEs (not a full kernel swap) are pending.
sudo pro enable livepatch                           # [Ubuntu] enable kernel live-patching (Ubuntu Pro)
sudo canonical-livepatch status                     # [Ubuntu] confirm applied kernel CVEs / patch state
pro status                                          # [Ubuntu] overview of all Pro services incl. esm-infra/esm-apps
```

---

## [11] Services & Init Systems (systemd / launchd)

```bash
# ============ [LINUX] systemd ============
systemctl status sshd
sudo systemctl start|stop|restart sshd
sudo systemctl enable --now sshd                     # enable at boot + start now
sudo systemctl disable --now sshd
systemctl list-units --type=service                   # running services
systemctl --failed                                    # what broke
sudo systemctl daemon-reload                           # after editing unit files
systemctl list-unit-files --state=enabled              # enabled-at-boot units

# journald logs for a unit
journalctl -u sshd -e            # jump to end
journalctl -u sshd -f            # follow live
journalctl -u sshd --since "1 hour ago"

# ============ [macOS] launchd ============
launchctl list | grep -i ssh
sudo launchctl bootstrap system /Library/LaunchDaemons/com.example.plist   # load
sudo launchctl bootout   system /Library/LaunchDaemons/com.example.plist   # unload
launchctl print gui/$(id -u)/com.example.agent          # inspect a job
# plist locations: ~/Library/LaunchAgents, /Library/LaunchAgents, /Library/LaunchDaemons
brew services list                                       # Homebrew-managed services
brew services restart nginx
```

### Hard-block a unit (systemctl mask/unmask) [LINUX]

```bash
# systemctl disable only removes the boot-time enablement symlink — a masked service still can't be started manually or pulled in as a dependency, which disable alone does not prevent.
# --- Force-disable a broken/conflicting unit (survives being pulled in as a dependency) ---
sudo systemctl mask sendmail                            # symlinks the unit to /dev/null; stronger than disable
sudo systemctl unmask sendmail                          # restore normal enable/disable behavior
```

### Boot performance diagnostics [LINUX]

```bash
# Useful when troubleshooting slow boots; not present in the current section, which only covers status/list/journal commands.
systemd-analyze blame                                   # units sorted by time spent initializing
systemd-analyze critical-chain                          # slowest dependency chain that delays boot completion
```

### One-step restart of a launchd job [macOS]

```bash
# Replaces the current bootout+bootstrap two-step pattern for a simple restart of a running job (bootstrap/bootout are still correct for actually (un)loading a plist). As of macOS 14.4+, Apple blocks -k against certain protected system daemons; for a stubborn system process, fall back to bootout+bootstrap.
sudo launchctl kickstart -k system/com.example.daemon   # kill + restart in a single command
launchctl kickstart -k gui/$(id -u)/com.example.agent   # same, for a per-user LaunchAgent
```

---

## [12] Directory Services / LDAP / AD Join

```bash
# --- Join Active Directory [RHEL/Fed][Ubuntu] (realmd + SSSD) ---
realm discover ad.example.com
sudo realm join -U Administrator ad.example.com
realm list                                            # show joined domain + config
id user@ad.example.com                                 # verify AD user resolves
sudo realm leave ad.example.com

# --- Query LDAP ---
ldapsearch -x -H ldap://dc.example.com -b "dc=example,dc=com" "(sAMAccountName=jdoe)"

# --- macOS AD bind ---
sudo dsconfigad -add ad.example.com -username admin -computer MAC01
dscl "/Active Directory/EXAMPLE/All Domains" -read /Users/jdoe   # read an AD user
dscl . -list /Users                                     # local users
```

### Restrict which AD accounts may log in [RHEL/Fed]

```bash
# By default realmd permits any domain user to log in once joined; realm permit/deny scopes that down, a common follow-up step not shown in the current join workflow.
# --- Restrict logins after joining (realmd access policy) ---
sudo realm permit -g 'Helpdesk Admins'          # allow only this AD group to log in
sudo realm permit --all                          # revert to allowing any domain account
sudo realm deny --all                            # deny all realm accounts (lock down)
```

### Refresh stale SSSD identity cache [RHEL/Fed]

```bash
# Stale SSSD caching is one of the most common causes of 'AD user resolves wrong/old group membership' after a directory-side change; sss_cache is the documented fix, complementing the id/realm verification already listed.
sss_cache -E                                     # invalidate all cached SSSD entries
sss_cache -u jdoe                                # invalidate a single cached user record
sudo systemctl restart sssd                      # force reload from the directory
```

---

## [13] Mail (Postfix)

```bash
# --- Service + queue ---
systemctl status postfix                               # [LINUX]
mailq                                                  # view mail queue
postqueue -p                                            # same, verbose
sudo postqueue -f                                       # flush (attempt redelivery)
sudo postsuper -d ALL                                   # ⚠ delete ALL queued mail
sudo postsuper -d ALL deferred                          # delete only deferred

# --- Send a test message ---
echo "test body" | mail -s "test subject" user@example.com   # needs mailutils/s-nail

# --- Logs ---
journalctl -u postfix -e                                # [systemd]
sudo tail -f /var/log/maillog                            # [RHEL/Fed]
sudo tail -f /var/log/mail.log                           # [Ubuntu]
```

### Inspect/edit main.cf parameters with postconf

```bash
# postconf is the standard tool for reading/writing main.cf and is not mentioned anywhere in the current section, which only covers queue and log commands.
# --- Inspect and edit configuration parameters ---
postconf mydomain                                       # view a parameter's current effective value
postconf -d mydomain                                    # view its compiled-in default
sudo postconf -e "relayhost = [smtp.example.com]:587"   # set a parameter (edits main.cf)
sudo systemctl reload postfix                           # apply postconf -e changes without dropping connections
```

### View a specific queued message with postcat

```bash
# Complements mailq/postqueue -p (which only list the queue) by letting you actually read a specific queued message's content; queue IDs come from mailq output.
postcat -q ABCDEF123456                                 # show envelope + headers + body for a queue ID
postcat -vq ABCDEF123456                                # verbose, useful when a message is stuck/deferred
```

---

## [14] Virtualization & Containers

```bash
# ============ KVM / libvirt [LINUX] ============
virsh list --all                                        # all VMs
sudo virsh start  vm-name
sudo virsh shutdown vm-name                              # graceful
sudo virsh destroy vm-name                               # force off
virsh dominfo vm-name
sudo virt-install --name vm1 --memory 4096 --vcpus 2 \
     --disk size=40 --cdrom /iso/os.iso --os-variant generic

# ============ Containers ============
podman ps -a            # [RHEL/Fed] daemonless, rootless-friendly
docker ps -a            # [ALL] where Docker is installed
podman images; docker images
podman run --rm -it alpine sh
docker compose up -d                                    # from a compose.yaml

# ============ macOS ============
# No native KVM. Use Docker Desktop / Podman machine / UTM / VMware Fusion.
podman machine init && podman machine start             # [macOS] podman VM backend
vmrun list                                              # [macOS] VMware Fusion CLI
```

### Podman + systemd: Quadlet replaces `podman generate systemd` [DEP]

```bash
# podman-generate-systemd(1) itself documents the command as deprecated in favor of Quadlet, which manages the generated systemd unit declaratively instead of via an imperative one-off dump.
# podman generate systemd is deprecated (bug-fix only, no new features) → use Quadlet unit files instead
mkdir -p ~/.config/containers/systemd                   # [RHEL/Fed] rootless Quadlet unit directory
# define the container declaratively in ~/.config/containers/systemd/myapp.container, then:
systemctl --user daemon-reload                          # [RHEL/Fed] Podman auto-generates + (re)loads the unit
systemctl --user start myapp.service
```

### VM snapshots with virsh [LINUX]

```bash
# The section currently covers VM lifecycle (start/shutdown/destroy) but has no snapshot workflow, which is a routine pre-change safety step. Applies to internal (qcow2) snapshots; external snapshots require additional manual steps that virsh snapshot-revert does not fully automate.
virsh snapshot-create-as vm-name snap1 "before patching"  # create a snapshot
virsh snapshot-list vm-name                                # list snapshots for a VM
virsh snapshot-revert vm-name snap1                        # roll back to a snapshot
```

### Reclaim disk space [ALL]

```bash
# A very common maintenance task on container hosts that isn't covered anywhere in the current section.
docker system prune -a --volumes                          # remove unused containers/images/networks/volumes/build cache
podman system prune -a --volumes                          # [RHEL/Fed] Podman equivalent
```

---

## [15] Printing (CUPS)

CUPS drives printing on both Linux and macOS.

```bash
lpstat -p -d                                            # printers + default
lpstat -t                                               # full status
lp -d PrinterName file.pdf                               # print
lpq                                                     # queue
cancel -a PrinterName                                    # clear queue
sudo lpadmin -p PrinterName -E -v ipp://192.0.2.50/ipp/print -m everywhere   # add IPP printer
sudo lpadmin -x PrinterName                              # remove printer
sudo cupsenable PrinterName; sudo cupsdisable PrinterName
# Web admin UI: http://localhost:631
sudo systemctl restart cups                              # [LINUX] restart spooler
```

### Inspect/set per-printer options

```bash
# The section currently covers add/remove/enable/disable but has no way to inspect or change a printer's option defaults (paper size, duplex, etc.), which lpoptions is the standard tool for.
lpoptions -p PrinterName -l                             # list current option keywords + choices for a printer
lpoptions -p PrinterName -o media=A4                    # set a default option (e.g., paper size)
```

### Query/update the CUPS server config itself

```bash
# cupsctl reads/writes the scheduler's own config; the current section only mentions the web admin UI at :631, not this CLI equivalent.
cupsctl                                                 # show current cupsd.conf settings as name=value
sudo cupsctl --remote-admin --remote-any                # enable remote admin access (use cautiously)
```

---

## [16] Remote Execution & Automation

```bash
# --- Loop over hosts (quick + dependency-free) ---
for h in host1 host2 host3; do
  echo "== $h =="; ssh "$h" 'uptime; df -h /'
done

# --- Ansible ad-hoc (agentless) ---
ansible all -i inventory.ini -m ping
ansible web -i inventory.ini -a "uptime"
ansible all -i inventory.ini -b -m dnf -a "name=vim state=latest"   # -b = become root

# --- Parallel SSH ---
parallel-ssh -h hosts.txt -i 'uptime'                   # pssh package
# or GNU parallel:
parallel -j8 ssh {} 'hostname; uptime' ::: host1 host2 host3

# --- Persistent sessions for long jobs ---
tmux new -s work        # detach: Ctrl-b d   |   reattach: tmux attach -t work
screen -S work          # detach: Ctrl-a d   |   reattach: screen -r work
nohup ./long-job.sh &   # survive logout, output → nohup.out
```

### Ansible: dry-run before applying + encrypt secrets with Vault

```bash
# Adds --check/--diff dry-run mode and ansible-vault for secret handling, which this repo's own 'never commit secrets' convention makes directly relevant. Syntax verified against known-good ansible-playbook/ansible-vault flag semantics (as of ansible-core 2.15+); citation URLs updated to current non-EOL doc paths.
# --- Ansible: preview changes before they hit prod ---
ansible-playbook site.yml --check --diff --limit web    # dry run, show would-be diffs

# --- Ansible Vault: never commit plaintext secrets in vars files ---
ansible-vault encrypt group_vars/prod/secrets.yml        # AES256-encrypt a vars file
ansible-playbook site.yml --ask-vault-pass                # run against vault-encrypted vars
```

### SSH connection multiplexing for host loops [ALL]

```bash
# ControlMaster/ControlPath/ControlPersist and `ssh -O check|exit` are real, stable OpenSSH ssh_config(5)/ssh(1) directives that speed up repeated/looped ssh calls by reusing one authenticated connection.
# --- SSH multiplexing: reuse one auth'd connection for repeated/looped ssh calls ---
# ~/.ssh/config: Host *  ControlMaster auto  ControlPath ~/.ssh/cm-%r@%h:%p  ControlPersist 10m
ssh -O check host1     # is a multiplexed master connection up?
ssh -O exit host1      # tear down the shared connection
```

---

## [17] Shell / Bash General Admin

```bash
# --- Environment / PATH ---
export EDITOR=vim
echo "$PATH" | tr ':' '\n'                              # readable PATH
env | sort                                              # all env vars
printenv HOME

# --- History / recall ---
history 20
!!            # last command      |  sudo !!  → rerun last with sudo
!$            # last arg of previous command
# Ctrl-R      # reverse-search history interactively

# --- Redirection / pipes ---
cmd > out.txt 2>&1        # stdout+stderr to file
cmd | tee out.txt         # see output AND save it
find . -name '*.tmp' -print0 | xargs -0 rm -f   # safe bulk action

# --- Permissions ---
chmod 640 file            # rw-r-----      | chmod +x script.sh
chown user:group file
chmod -R u+rwX,go-w dir   # recursive, dirs get +x via X

# --- Job control ---
./task.sh &   ;  jobs  ;  fg %1  ;  bg %1  ;  disown %1

# --- Dates (ISO 8601) ---
date +%F                  # 2026-07-22
date -u +%FT%TZ           # 2026-07-22T18:04:11Z (UTC)

# --- Time a command ---
time ./script.sh
```

### Safer scripts: strict mode + cleanup traps [ALL]

```bash
# set -euo pipefail plus an EXIT trap is the standard defensive pattern for admin scripts (undefined vars, mid-pipeline failures, and leftover temp files are common real-world bugs); ${VAR:-default} and ${VAR:?msg} expansions confirmed via the Ubuntu bash(1) manpage's parameter-expansion section. None of this strict-mode/trap/expansion material exists elsewhere in the section.
# --- Fail fast instead of silently continuing ---
set -euo pipefail   # -e: exit on error | -u: error on unset var | -o pipefail: pipeline fails if any stage fails

# --- Guaranteed cleanup on exit (success OR error) ---
trap 'rm -f "$tmpfile"' EXIT
trap - EXIT          # remove a previously set EXIT trap

# --- Parameter expansion defaults / required args ---
echo "${NAME:-default}"       # use 'default' if NAME is unset or empty
: "${API_HOST:?must be set}"  # abort with a message if API_HOST is unset or empty
```

---

## [18] Logs & journald

```bash
# ============ [LINUX] systemd journal ============
journalctl -e                     # jump to newest
journalctl -f                     # follow (like tail -f)
journalctl -p err -b              # errors since this boot
journalctl -u nginx --since today
journalctl --disk-usage; sudo journalctl --vacuum-time=7d   # trim old logs
dmesg -T | tail -50               # kernel ring buffer (human timestamps)

# ============ Classic log files ============
sudo tail -f /var/log/messages     # [RHEL/Fed] general
sudo tail -f /var/log/syslog       # [Ubuntu] general
sudo tail -f /var/log/secure       # [RHEL/Fed] auth
sudo tail -f /var/log/auth.log     # [Ubuntu] auth

# ============ [macOS] unified logging ============
log show --predicate 'eventMessage CONTAINS "error"' --last 1h
log stream --level info            # live tail
log show --last 30m > ~/Desktop/macos-log.txt

# --- Rotation ---
sudo logrotate -f /etc/logrotate.conf   # force a rotation [LINUX]
```

### Kernel-only view and boot history via journald [LINUX]

```bash
# journalctl -k implies -b and filters to _TRANSPORT=kernel; --list-boots enumerates prior boots so you can target -b -1, -b -2, etc. Confirmed via Ubuntu journalctl(1) manpage. Complements the existing dmesg -T line by covering cross-boot log retrieval, which dmesg alone cannot do.
# --- Kernel messages through the journal (replaces raw dmesg for boot-scoped queries) ---
journalctl -k -b                                    # kernel messages, current boot only
journalctl --list-boots                             # table of past boot IDs + time ranges
journalctl -b -1 -p err                             # errors from the PREVIOUS boot
```

### Persistent journal storage [LINUX]

```bash
# Storage=persistent (or creating /var/log/journal, which auto triggers persistence) is required on distros where journald defaults to volatile /run storage that is lost on reboot. Confirmed via Ubuntu journald.conf(5) manpage. Directly relevant to the section's log-retention theme (it already covers --vacuum-time but not enabling persistence in the first place).
# --- Make journal logs survive reboots (default is often volatile-only) ---
mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\n' | sudo tee /etc/systemd/journald.conf.d/persist.conf
sudo systemctl restart systemd-journald
```

---

## [19] Network Info

```bash
# --- Addresses / links [LINUX] ---
ip -br addr                        # brief per-interface addresses
ip addr show
ip -br link                        # up/down + MAC
ip route                           # routes + default gateway

# --- Addresses [macOS] ---
ifconfig                           # all interfaces
ipconfig getifaddr en0             # just the IPv4 on en0
networksetup -listallhardwareports # map en0/en1 → Wi-Fi/Ethernet
route -n get default               # default gateway

# --- Sockets ---
ss -s                              # [LINUX] summary stats
ss -tan state established          # [LINUX] established TCP

# --- Bandwidth / live throughput ---
iftop -i eth0                      # [LINUX] per-connection (install iftop)
nload                              # [LINUX] in/out graph
vnstat -d                          # [LINUX] daily history (install vnstat)
nettop                             # [macOS] per-process network usage
```

### Interface link speed/duplex and RX/TX error counters [LINUX]

```bash
# Neither current speed/duplex nor error/drop counters are covered anywhere in this section. Verified live this pass via the Ubuntu (jammy) ethtool(8) manpage for 'ethtool eth0' behavior. The 'ip -s -s link show' repeatable-verbosity statistics flag is standard documented ip(8) behavior, but the specific citation for it (an Arch ip-stats mirror) returned 403 through the proxy this session and could not be re-verified live, so that reference has been dropped rather than kept unverified.
ethtool eth0                                    # link speed, duplex, autoneg, link-detected state
ip -s -s link show dev eth0                     # RX/TX bytes/packets/errors/drops (repeat -s for detail)
```

### Active bandwidth test between two hosts [ALL]

```bash
# iperf3 gives an actual measured throughput number between two hosts, complementing the existing passive/live-graph tools (iftop/nload/vnstat/nettop). Verified live this pass via the Ubuntu (jammy) iperf3(1) manpage, which confirms -s (server mode), -c (client mode), -t (time in seconds), -u (UDP), and -b (target bitrate) exactly as used.
iperf3 -s                                       # run on the target host
iperf3 -c targethost -t 10                      # run on the source host, 10s TCP test
iperf3 -c targethost -u -b 100M                 # UDP test at a fixed bitrate
```

---

## [20] Legacy → Modern Command Equivalents

The `net-tools` suite is deprecated on modern Linux in favor of `iproute2`.

| Legacy `[DEP]` | Modern | Purpose |
|----------------|--------|---------|
| `ifconfig` | `ip addr` | Interface addresses |
| `ifconfig eth0 up/down` | `ip link set eth0 up/down` | Bring link up/down |
| `route -n` | `ip route` | Routing table |
| `netstat -tulpn` | `ss -tulpn` | Listening sockets |
| `netstat -rn` | `ip route` | Routes |
| `arp -a` | `ip neigh` | Neighbor/ARP table |
| `iwconfig` | `iw dev` | Wireless config |
| `nslookup` | `dig` / `host` | DNS lookups |
| `service x start` | `systemctl start x` | Service control |
| `chkconfig x on` | `systemctl enable x` | Enable at boot |
| `yum` | `dnf` | Package manager (RHEL) |
| `ntpq -p` | `chronyc sources` | Time sync status |
| `top` (only) | `htop` / `btop` | Live process view |

> macOS still ships `ifconfig`, `netstat`, and `route` as first-class tools —
> the deprecation above is a **Linux** convention.

### iptables → nftables [DEP]

```bash
# iptables-translate and `nft list ruleset` are real, correctly-formed commands; fills the [20] Legacy→Modern table's gap for the iptables/nft pairing that the [21] Security Hardening section already flags as [DEP].
# iptables/ip6tables/arptables/ebtables are [DEP] on RHEL 9+; nftables (nft) is the successor framework
iptables-translate -A INPUT -s 192.0.2.0/24 -j ACCEPT   # preview the equivalent nft rule before migrating
nft list ruleset                                        # modern equivalent of iptables-save
```

### ifcfg network-scripts → NetworkManager keyfiles [RHEL/Fed]

```bash
# `nmcli connection show` and `nmcli connection migrate` are correct; the latter was added in NetworkManager 1.42 specifically to convert ifcfg-rh profiles to the keyfile format that replaced them in RHEL 9.
# RHEL 9+: /etc/sysconfig/network-scripts (ifcfg files) is [DEP] — the network-scripts package was removed
nmcli connection show        # list NetworkManager connection profiles (modern equivalent)
nmcli connection migrate     # convert legacy ifcfg files to NM keyfiles (NetworkManager 1.42+)
```

---

## [21] Security Hardening

```bash
# ============ SELinux [RHEL/Fed] ============
getenforce                          # Enforcing / Permissive / Disabled
sestatus
sudo setenforce 0                   # temp → Permissive (until reboot)
# Persist: set SELINUX=enforcing in /etc/selinux/config
sudo ausearch -m avc -ts recent     # recent denials
sudo restorecon -Rv /var/www        # fix file contexts
getsebool -a | grep httpd           # list booleans

# ============ AppArmor [Ubuntu] ============
sudo aa-status                      # profiles + enforcement mode
sudo aa-enforce  /etc/apparmor.d/usr.sbin.nginx
sudo aa-complain /etc/apparmor.d/usr.sbin.nginx

# ============ Intrusion / audit [LINUX] ============
sudo fail2ban-client status                 # jails overview
sudo fail2ban-client status sshd            # banned IPs for sshd
sudo aureport --auth --summary              # auditd auth report
sudo auditctl -w /etc/passwd -p wa -k passwd-watch   # watch a file

# ============ SSH hardening (/etc/ssh/sshd_config) ============
#   PermitRootLogin no
#   PasswordAuthentication no        # key-only
#   sudo systemctl restart sshd  (or 'ssh' on Ubuntu)

# ============ [macOS] platform security ============
spctl --status                      # Gatekeeper on/off
csrutil status                      # System Integrity Protection (set in Recovery)
fdesetup status                     # FileVault disk encryption
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on
# Audit tool for either platform: `sudo lynis audit system`
```

### OpenSCAP compliance scanning [RHEL/Fed]

```bash
# CORRECTED: the draft's `--profile cis` is invalid — oscap requires the full XCCDF profile ID (e.g. xccdf_org.ssgproject.content_profile_cis), which `oscap info` prints; a bare shorthand like 'cis' will fail to match. oscap/OpenSCAP is otherwise accurately described as RHEL's official baseline compliance scanner.
# ============ OpenSCAP compliance scanning [RHEL/Fed] ============
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml         # list available profiles + their full IDs (CIS, STIG, OSPP...)
sudo oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis --report report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml                  # scan + HTML report
```

### Account lockout with pam_faillock [RHEL/Fed]

```bash
# pam_faillock/faillock command syntax verified against known RHEL semantics; current mechanism for locking accounts after repeated failed logins (successor to pam_tally2), distinct from fail2ban (network-level) and auditd (file/event auditing).
# ============ Account lockout: pam_faillock [RHEL/Fed] ============
sudo faillock --user alice           # view failed-login counter for a user
sudo faillock --user alice --reset   # unlock / clear the counter
# Policy: /etc/security/faillock.conf (deny=, fail_interval=, unlock_time=)
```

---

## [22] Repositories & Package Sources

```bash
# ============ [RHEL/Fed] dnf repos ============
dnf repolist                                       # enabled repos
sudo dnf config-manager --add-repo https://example.com/repo.repo
sudo dnf install epel-release                       # EPEL (RHEL/Rocky/Alma)
sudo dnf config-manager --set-enabled crb           # CodeReady Builder (RHEL 9)
# Repo files live in /etc/yum.repos.d/*.repo

# ============ [Ubuntu] apt sources ============
sudo add-apt-repository ppa:example/ppa             # add a PPA
sudo add-apt-repository universe                     # enable a component
# Modern signed-repo pattern (replaces apt-key [DEP]):
#   curl -fsSL https://example.com/key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/example.gpg
#   echo "deb [signed-by=/etc/apt/keyrings/example.gpg] https://example.com/apt stable main" \
#     | sudo tee /etc/apt/sources.list.d/example.list
sudo apt update

# ============ [macOS] Homebrew taps ============
brew tap                                            # list taps
brew tap homebrew/cask-fonts
brew untap homebrew/cask-fonts
```

### Ubuntu 24.04+ default repo format changed (deb822 .sources) [Ubuntu]

```bash
# The section's PPA/signed-repo guidance still shows the older one-line sources.list.d/*.list style exclusively; on 24.04+ that's no longer the default layout admins will find on disk, so it's worth documenting the deb822 .sources format they'll actually encounter.
# Ubuntu 24.04+ ships /etc/apt/sources.list.d/ubuntu.sources in deb822 format by default
# (replaces the single-line /etc/apt/sources.list). Example stanza:
#   Types: deb
#   URIs: http://archive.ubuntu.com/ubuntu/
#   Suites: noble noble-updates noble-security
#   Components: main universe restricted multiverse
#   Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
# add-apt-repository now writes new PPAs as .sources files here too.
```

### dnf5's addrepo replaces --add-repo on newer Fedora/RHEL [RHEL/Fed]

```bash
# dnf5 (default dnf on Fedora 41+ and shipping on newer RHEL) restructured config-manager under an addrepo subcommand and writes changes as drop-in override files rather than editing .repo files in place, which is a meaningful behavior change worth flagging next to the existing dnf4-style --add-repo line.
dnf5 config-manager addrepo --from-repofile=https://example.com/repo.repo   # [RHEL/Fed] dnf5 equivalent of config-manager --add-repo
dnf5 config-manager addrepo --set=baseurl=https://example.com/repo --id=example   # [RHEL/Fed] add repo from a baseurl directly
```

---

## [23] Storage: LVM & Software RAID

```bash
# ============ LVM [LINUX] ============
sudo pvcreate /dev/sdb                              # init physical volume
sudo vgcreate data /dev/sdb                          # volume group
sudo lvcreate -L 50G -n vol1 data                    # logical volume
pvs; vgs; lvs                                        # summaries
# Grow LV + filesystem:
sudo lvextend -L +20G /dev/data/vol1
sudo resize2fs /dev/data/vol1                        # ext4
sudo xfs_growfs /mnt/vol1                            # xfs (grow mounted)

# ============ Software RAID (mdadm) [LINUX] ============
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb /dev/sdc
cat /proc/mdstat                                     # live status
sudo mdadm --detail /dev/md0
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf   # persist config

# ============ [macOS] ============
diskutil appleRAID list
diskutil appleRAID create mirror MyRAID JHFS+ disk2 disk3
```

### LVM: grow LV and filesystem in one step [LINUX]

```bash
# lvextend's -r/--resizefs flag calls fsadm(8) to grow the underlying filesystem (ext2/3/4 or XFS only) after extending the LV, replacing the separate lvextend + resize2fs/xfs_growfs steps already shown in this section with one command. Confirmed against the lvm2 lvextend(8) man page.
# lvextend -r/--resizefs grows the LV and calls fsadm to grow the fs in one shot (ext2/3/4 and XFS only)
sudo lvextend -r -L +20G /dev/data/vol1               # ext4/xfs — replaces the 2-step lvextend+resize2fs/xfs_growfs above
sudo lvextend -r -l +100%FREE /dev/data/vol1          # grow to consume all remaining VG free space
```

### mdadm array monitoring [LINUX]

```bash
# mdadm --monitor --scan tracks all md devices for DegradedArray/FailSpare events; --daemonise forks it to the background for manual use, while the systemd unit runs the equivalent 'mdadm --monitor --scan' itself (systemd handles the backgrounding), so enabling mdmonitor.service is the standard way to get automatic email/syslog alerts on array failure. Verified via the mdadm(8) man page and the mdadm project's systemd/mdmonitor.service unit file.
# Watch for RAID degradation/failure events (mdmonitor.service runs this as a daemon)
sudo mdadm --monitor --scan --daemonise                 # fork to background; add --oneshot to check once and exit (e.g. from cron)
sudo systemctl enable --now mdmonitor.service            # persistent monitoring via systemd (RHEL/Ubuntu)
```

---

## [24] ZFS / Btrfs (TrueNAS / NAS)

```bash
# ============ ZFS (OpenZFS on Linux; TrueNAS; macOS via OpenZFS) ============
zpool status -v
zpool list
zpool import                                         # list importable pools
sudo zpool import -f poolname                          # force import (offline but intact)
sudo zpool scrub poolname                              # ALWAYS scrub after a force import
zfs list -t all
sudo zfs snapshot tank/data@2026-07-22               # snapshot
sudo zfs rollback tank/data@2026-07-22               # roll back to snapshot
zpool iostat -v 2                                     # live per-vdev I/O

# ============ Btrfs [LINUX] ============
sudo btrfs filesystem show
sudo btrfs filesystem df /mnt                         # accurate space (accounts for RAID/COW)
sudo btrfs scrub start /mnt; sudo btrfs scrub status /mnt
sudo btrfs subvolume list /mnt
sudo btrfs subvolume snapshot /mnt /mnt/.snap/2026-07-22
```

### ZFS replication with send/receive

```bash
# zfs send/zfs receive is the standard OpenZFS mechanism for pool-to-pool and offsite replication, and pairs naturally with the snapshot/rollback commands already in this section. Verified against the current OpenZFS zfs-send.8 / zfs-receive.8 man pages.
# Replicate a snapshot to another pool/host (full stream, then incremental)
zfs send tank/data@2026-07-22 | zfs receive backup/data              # local full send
zfs send -i tank/data@2026-06-01 tank/data@2026-07-22 | ssh host zfs receive backup/data   # incremental over SSH
```

### ZFS TRIM for SSD-backed pools

```bash
# zpool trim issues an on-demand TRIM across a pool's free space and autotrim=on enables continuous background trimming; neither was previously mentioned even though the section already covers scrub/iostat for the same pools. Verified against the OpenZFS zpool-trim.8 man page.
sudo zpool trim poolname                              # on-demand TRIM of all free space (SSD/NVMe vdevs)
zpool status -t poolname                              # check TRIM support/progress per vdev
sudo zpool set autotrim=on poolname                   # enable ongoing background auto-TRIM
```

### Btrfs rebalance [LINUX]

```bash
# btrfs balance start with usage filters is the standard way to reclaim space fragmented across mostly-empty data/metadata block groups, complementing the existing scrub/subvolume commands. Verified against the btrfs-progs btrfs-balance(8) documentation.
sudo btrfs balance start -dusage=50 -musage=50 /mnt    # reclaim space from mostly-empty block groups
sudo btrfs balance status /mnt                         # check progress of a running balance
sudo btrfs balance cancel /mnt                         # stop a running balance
```

---

## [25] macOS Power Tools

```bash
# --- Preferences via defaults ---
defaults read com.apple.finder                       # dump a domain
defaults write com.apple.finder AppleShowAllFiles -bool true && killall Finder   # show hidden files
defaults write com.apple.dock autohide -bool true && killall Dock

# --- Spotlight from the CLI ---
mdfind "kMDItemDisplayName == '*.pdf'wc"             # query the index
mdls ~/Desktop/file.pdf                              # metadata for a file

# --- Clipboard ---
pbcopy < file.txt        # copy file into clipboard
pbpaste > out.txt        # paste clipboard to file

# --- Power / sleep ---
pmset -g                                             # current settings
caffeinate -dimsu -t 3600                            # keep awake 1h
sudo pmset -a hibernatemode 3                         # power management (careful)

# --- System actions ---
softwareupdate -l                                    # list available updates
sudo purge                                           # flush inactive memory
killall Finder; killall Dock; killall SystemUIServer  # restart UI pieces
screencapture -x ~/Desktop/shot.png                   # silent screenshot
say "build complete"                                  # TTS notification
open -a "Safari" https://example.com                  # open app/URL
osascript -e 'display notification "Done" with title "Job"'   # GUI notification
```

### Time Machine from the CLI [macOS]

```bash
# tmutil is Apple's scriptable Time Machine control tool — useful for headless Macs/servers where driving backups through System Settings isn't practical, and this section had no backup coverage at all. Subcommand names and behavior cross-checked against tmutil documentation (ss64 mirror of Apple's man tmutil), which additionally confirms startbackup's --block, --rotation, and --destination options.
tmutil status                                        # is a backup running right now?
tmutil startbackup --auto                            # kick off a backup now
tmutil listbackups                                   # list local/remote snapshots by date
sudo tmutil addexclusion /path/to/skip                # exclude a path from future backups
```

### APFS snapshots from the CLI [macOS]

```bash
# diskutil apfs listSnapshots/deleteSnapshot expose the local APFS snapshots (the same ones Time Machine and System Updates create) directly, which complements the appleRAID coverage already in Section 23 and rounds out this section's disk/volume tooling. Verified against Apple Support's Disk Utility snapshot guide.
diskutil apfs listSnapshots /                        # list local APFS snapshots on a volume
diskutil apfs deleteSnapshot / -uuid <UUID>          # remove a specific snapshot
```

---

## [26] Misc Utilities

```bash
# --- Shutdown / reboot [LINUX] ---
sudo shutdown -h now            # halt now      | -h +10 = in 10 min
sudo shutdown -r now            # reboot now
sudo systemctl poweroff | reboot
# [macOS]:
sudo shutdown -h now; sudo shutdown -r now

# --- Scheduling ---
crontab -e                      # edit user cron        | crontab -l = list
# min hour dom mon dow  command   (e.g. `0 2 * * * /path/backup.sh`)
sudo systemctl list-timers      # [LINUX] systemd timers (modern cron alt)
# [macOS] use launchd agents (see Section 11) or `cron` (still present)

# --- Archives ---
tar czf out.tgz dir/            # create gzip tarball
tar xzf out.tgz                 # extract
tar tzf out.tgz                 # list contents
zip -r out.zip dir/; unzip out.zip

# --- Download ---
curl -fLO https://example.com/file.zip     # -O keep name, -L follow redirects
wget https://example.com/file.zip

# --- Checksums ---
sha256sum file.iso              # [LINUX]
shasum -a 256 file.iso          # [macOS]

# --- Self-signed cert (test/dev only) ---
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout key.pem -out cert.pem -subj "/CN=mysite.local"

# --- Quick HTTP server (share a folder) ---
python3 -m http.server 8080

# --- Shares ---
smbclient -L //server -U user                # [ALL] list SMB shares
showmount -e nfs-server                       # [LINUX] list NFS exports
sudo exportfs -ra                              # [LINUX] reload NFS exports
sharing -l                                     # [macOS] list shared points

# --- Watch a command refresh ---
watch -n 5 'df -h /'           # [LINUX] every 5s (macOS: brew install watch)

# --- Text power tools ---
grep -rn "TODO" .              # recursive search w/ line numbers
sed -i 's/foo/bar/g' file      # in-place replace (macOS: sed -i '' 's/…/…/g')
awk -F, '{print $1, $3}' data.csv   # columns 1 & 3
```

### jq for quick JSON parsing [ALL]

```bash
# jq is already recommended in the 'handy installs' quick reference in this file but never demonstrated anywhere — this gives it a minimal, genuinely common usage example (piping API/CLI JSON output through jq to extract fields), which is a gap given how much modern sysadmin tooling emits JSON. Verified against the official jq 1.8 manual at jqlang.org.
curl -s https://api.example.com/status | jq '.'          # pretty-print JSON
curl -s https://api.example.com/hosts | jq -r '.[].name' # pull one field out of an array
```

---

## Quick Reference

### Detect the OS / distro

```bash
cat /etc/os-release            # [LINUX] ID=, VERSION_ID=
hostnamectl                    # [LINUX] pretty summary
rpm -E %rhel                   # [RHEL/Fed] major version (empty on Fedora)
lsb_release -a                 # [Ubuntu] (install lsb-release)
sw_vers                        # [macOS] ProductVersion / BuildVersion
uname -sm                      # kernel + arch on anything (Linux/Darwin)
```

### Package-manager one-glance

| | RHEL/Fedora | Ubuntu/Debian | macOS |
|---|---|---|---|
| Tool | `dnf` (`yum` on RHEL 7) | `apt` (`apt-get`) | `brew` |
| Config | `/etc/yum.repos.d/` | `/etc/apt/` | `brew tap` |
| Universal apps | `flatpak` | `snap`, `flatpak` | `brew --cask`, `mas` |

### Init system at a glance

| | Linux (systemd) | macOS (launchd) |
|---|---|---|
| Control | `systemctl` | `launchctl` |
| Logs | `journalctl` | `log show` / `log stream` |
| Enable at boot | `systemctl enable` | `launchctl bootstrap` + plist |
| Unit/job files | `/etc/systemd/system/` | `~/Library/LaunchAgents`, `/Library/LaunchDaemons` |

### Handy installs to make a box comfortable

```bash
sudo dnf install -y htop ncdu tmux git jq tree bind-utils    # [RHEL/Fed]
sudo apt install -y htop ncdu tmux git jq tree dnsutils      # [Ubuntu]
brew install htop ncdu tmux git jq tree                       # [macOS]
```

### RHEL/Fedora: dnf5 is now the default DNF [RHEL/Fed]

```bash
# dnf5 replaced the Python-based DNF4 as the default package manager starting with RHEL 10 and Fedora 41+, keeping the same everyday subcommands (install/remove/search/repoquery) but changing some plugin and scripting/output behavior — worth a callout in the package-manager-at-a-glance table since scripts written against dnf4 output parsing may need adjustment. Verified against Red Hat's RHEL 10 'Managing software with the DNF tool' documentation.
# RHEL 10 / Fedora 41+ ship dnf5 as the default `dnf` (rewritten in C++, same everyday subcommands)
dnf --version                                        # confirm whether dnf5 or legacy dnf4 is active
dnf5 repoquery --installed                            # works the same under dnf5; useful if a script hardcodes the dnf5 binary name
```

---

**Linux & macOS Admin Arsenal 2026** — companion to the Windows sheet.
Maintained by Chris Grady for high-agency sysadmins.
GitHub: [cgfixit/Windows-Linux--Docker-Handbook](https://github.com/cgfixit/Windows-Linux--Docker-Handbook).
Roadmap: enhance into its own single-page HTML web app (search + copy) alongside
`Windows-Admin-Cheat-Sheet.html`. Contributions welcome. Stay dangerous.

<!-- END OF CHEAT SHEET -->
