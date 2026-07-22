# Windows - Linux - MacOS - Docker/Containers Arsenal 2026

One handbook, four platform families. Curated, security-conscious one-liners for
modern system administration, troubleshooting, automation, and hardening —
maintained by Chris Grady ([@cgfixit](https://github.com/cgfixit)).

The Windows and Linux/macOS sheets are plain Markdown, each backed by a
**single-page web app** (instant search, per-command copy buttons, printable) —
one HTML file, no build step, no tracking. Styling loads from CDN
Tailwind/Font Awesome, so first load wants internet; everything else is local.

## 🚀 Use it now

| Platform | Reference (Markdown) | Interactive web app |
|----------|---------------------|---------------------|
| 🪟 **Windows** — Server 2016/2019/2022/2025, Win10/11 | [`Windows-Admin-Cheat-Sheet.md`](./Windows-Admin-Cheat-Sheet.md) | [Live app](https://cgfixit.github.io/Windows-Linux--Docker-Handbook/Windows-Admin-Cheat-Sheet.html) |
| 🐧🍎 **Linux & macOS** — RHEL/Rocky/Alma, Fedora, Ubuntu/Debian, macOS | [`Linux-Mac-Admin-Cheat-Sheet.md`](./Linux-Mac-Admin-Cheat-Sheet.md) | [Live app](https://cgfixit.github.io/Windows-Linux--Docker-Handbook/Linux-Mac-Admin-Cheat-Sheet.html) |
| 🐳 **Docker / Containers** — Engine, CLI, Compose v2, Buildx | [`Docker-Containers.md`](./Docker-Containers.md) | *skeleton — web app coming* |

The Windows and Linux/macOS sheets each carry **26 numbered sections plus a
quick-reference appendix**, mirrored in structure so the two stay parallel.
The Docker sheet is an intentionally concise 13-section skeleton that will
grow into a full sheet + web app.

## ⚡ A taste of what's inside

**Windows** (CMD / PowerShell / CIM)

| Task | Command |
|------|---------|
| Which process owns port 443 | `Get-Process -Id (Get-NetTCPConnection -LocalPort 443).OwningProcess` |
| Test a remote port (no telnet) | `Test-NetConnection -ComputerName hostname -Port 443` |
| System file repair (in this order) | `DISM /Online /Cleanup-Image /RestoreHealth` → `sfc /scannow` |
| Update everything | `winget upgrade --all` |
| Hardware + OS inventory | `Get-ComputerInfo` |

**Linux** (RHEL / Fedora / Ubuntu)

| Task | Command |
|------|---------|
| Listening sockets + owning process | `ss -tulpn` |
| Errors since boot | `journalctl -p err -b` |
| Enable + start a service | `sudo systemctl enable --now sshd` *(Ubuntu: unit is `ssh`)* |
| Interface addresses, briefly | `ip -br addr` |
| Patch the box | `sudo dnf upgrade` / `sudo apt update && sudo apt full-upgrade` |

**macOS**

| Task | Command |
|------|---------|
| Flush DNS cache | `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` |
| Built-in speed/responsiveness test | `networkQuality -v` |
| Apply all Apple updates | `sudo softwareupdate -ia` |
| Keep the Mac awake for 1h | `caffeinate -dimsu -t 3600` |
| OS + build version | `sw_vers` |

**Docker**

| Task | Command |
|------|---------|
| What's running (or crashed) | `docker ps -a` |
| Bring up a stack | `docker compose up -d` |
| Shell into a container | `docker exec -it web sh` |
| Follow logs | `docker logs -f web` |
| Reclaim disk space | `docker system prune` |

## 🧭 Conventions (all sheets)

- **Never any secrets.** No passwords, keys, tokens, or private hostnames —
  placeholders only (`XXXXX`, `hostname`, `192.0.2.x`). Commands read secrets
  from interactive prompts, a vault, or env vars.
- **Compatibility tags** on every command:
  - Windows: `[ALL]` `[2019+]` `[2022+]` `[2025]` `[DEP]`
  - Linux/Mac: `[ALL]` `[LINUX]` `[RHEL/Fed]` `[Ubuntu]` `[macOS]` `[DEP]`
  - Docker: `[ALL]` `[LINUX]` `[DESKTOP]` `[DEP]`
- **Modern over legacy**, with the replacement shown next to anything
  deprecated (`wmic` → `Get-CimInstance`, `ifconfig` → `ip addr`,
  `netstat` → `ss`, `docker-compose` → `docker compose`).
- **Accuracy is the product.** The 2026 enhancement pass added 100+ commands
  researched against official documentation (Microsoft Learn, man pages,
  Red Hat/Ubuntu docs, Apple, Postfix/CUPS/libvirt/Docker/OpenZFS), with every
  citation independently verified before merge — including corrections where
  reality had moved (e.g. WMIC now ships *disabled by default* on Win11
  23H2/24H2 and Server 2025, with full removal on 25H2 — not merely deprecated).

## 🛠 Repo layout

Markdown is the source of truth; each web app is generated from its sheet and
committed alongside it. CI runs HTMLHint over the apps as a quality gate.
`CLAUDE.md` documents the conventions for AI-assisted maintenance.

> **Note:** this repository was renamed from *Windows-Admin-Cheat-Sheet* to
> **Windows-Linux--Docker-Handbook** as coverage grew. File names inside the
> repo keep their original platform-specific names.

---

**Windows - Linux - MacOS - Docker/Containers Arsenal 2026** — maintained by
Chris Grady for high-agency sysadmins.  
GitHub: [cgfixit/Windows-Linux--Docker-Handbook](https://github.com/cgfixit/Windows-Linux--Docker-Handbook)  
Live: <https://cgfixit.github.io/Windows-Linux--Docker-Handbook/>  
Contributions welcome. Stay dangerous.
