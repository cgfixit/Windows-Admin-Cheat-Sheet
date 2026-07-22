# Windows Admin Arsenal 2026

Your comprehensive cheat sheet for modern Windows administration, troubleshooting, automation, and security.  
Updated for Windows 11 24H2 / Server 2025 era. Curated by Chris Grady (@cgfixit)

> 🐧🍎🐳 **Cross-platform?** See the companion [**Linux & macOS Admin Cheat Sheet**](./Linux-Mac-Admin-Cheat-Sheet.md) — RHEL, Ubuntu, Fedora, and macOS one-liners, mirroring this sheet's structure — and the [**Docker & Containers Handbook**](./Docker-Containers.md) skeleton.

## 🔍 Quick PowerShell One-Liners

| Task                        | Command                                                                 |
|-----------------------------|-------------------------------------------------------------------------|
| List all installed software | `Get-WmiObject -Class Win32_Product \| Select Name, Version`             |
| Check disk space            | `Get-PSDrive -PSProvider FileSystem`                                    |
| Restart service             | `Restart-Service -Name "Spooler"`                                      |
| Export event logs           | `Get-EventLog -LogName System -Newest 100 \| Export-Csv events.csv`     |
| Find large files            | `Get-ChildItem -Path C:\\ -Recurse -ErrorAction SilentlyContinue \| Sort-Object Length -Descending \| Select-Object -First 20` |

## 💻 System Info & Diagnostics

- **Full system info:** `systeminfo`
- **Hardware details:** `Get-ComputerInfo`
- **BIOS info:** `Get-WmiObject win32_bios`
- **Driver versions:** `Get-WmiObject Win32_PnPSignedDriver \| Select FriendlyName, DriverVersion`
- **Check for updates:** `usoclient StartScan` or PowerShell: `Get-WindowsUpdate` (requires module)

## 🛡️ Security & Compliance

- Enable Windows Defender full scan: `Start-MpScan -ScanType FullScan`
- Check BitLocker status: `Get-BitLockerVolume`
- Reset local admin password: Use `net user administrator NewPass123!`
- YARA scanning integration (your custom scripts): Place in PATH and run `yara64.exe -r rules.yar C:\\`
- Event log security audit: `wevtutil qe Security /f:text`

**Pro Tip:** Integrate with your CyClaw/PsyClaw RAG for automated YARA rule generation and health checks.

## 📊 Performance & Monitoring

| Tool/Command                  | Use Case                        |
|-------------------------------|---------------------------------|
| Resource Monitor              | `resmon`                        |
| Performance Monitor           | `perfmon`                       |
| Process Explorer (Sysinternals) | Deep process analysis         |
| Check CPU/RAM usage           | `Get-Counter '\Processor(*)\% Processor Time'` |

## 🔧 Troubleshooting Common Issues

- SFC scan: `sfc /scannow`
- DISM repair: `DISM /Online /Cleanup-Image /RestoreHealth`
- Network reset: `netsh int ip reset` + `netsh winsock reset`
- Clear DNS cache: `ipconfig /flushdns`
- WinRE repair: Boot to recovery or `reagentc /enable`

## ⚙️ Automation & Scripting

- Batch rename files: PowerShell foreach loop
- Scheduled tasks: `schtasks` or `New-ScheduledTask`
- Group Policy refresh: `gpupdate /force`
- AD queries: `Get-ADUser` (RSAT tools)
- Veeam-specific: Your custom PowerShell modules for backups/health checks

## 📁 Useful Paths & Tools

- Event Viewer: `eventvwr.msc`
- Services: `services.msc`
- Device Manager: `devmgmt.msc`
- Sysinternals Suite: Always keep updated
- Windows Admin Center: Modern web-based management

---

**Windows Admin Arsenal 2026** — Maintained by Chris Grady for high-agency sysadmins.  
GitHub: [CGFixIT/Windows-Linux--Docker-Handbook](https://github.com/CGFixIT/Windows-Linux--Docker-Handbook)  
Live: https://cgfixit.github.io/Windows-Linux--Docker-Handbook/  
Contributions welcome. Stay dangerous.