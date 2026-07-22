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

---

**Linux & macOS Admin Arsenal 2026** — companion to the Windows sheet.
Maintained by Chris Grady for high-agency sysadmins.
GitHub: [cgfixit/Windows-Admin-Cheat-Sheet](https://github.com/cgfixit/Windows-Admin-Cheat-Sheet).
Roadmap: enhance into its own single-page HTML web app (search + copy) alongside
`Windows-Admin-Cheat-Sheet.html`. Contributions welcome. Stay dangerous.

<!-- END OF CHEAT SHEET -->
