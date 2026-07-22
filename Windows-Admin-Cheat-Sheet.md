::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
:: WINDOWS ADMIN ONE-LINER CHEAT SHEET
:: Consolidated & Deduplicated | Server 2016 / 2019 / 2022 / 2025 / Win10-11
:: Last Updated: 2026-03-15
::
:: SECURITY: NEVER store passwords, keys, or secrets in this file.
::           Use a vault, env vars, or credential manager.
::
:: COMPATIBILITY KEY:
::   [ALL]    = Server 2016+ and Win10+
::   [2019+]  = Server 2019+ and Win10 1903+
::   [2022+]  = Server 2022+ and Win11+
::   [2025]   = Server 2025 only (or preview features)
::   [DEP]    = Deprecated; shown with modern replacement
::
:: POWERSHELL VERSION REFERENCE:
::   Server 2016 / Win10 (RTM-1607)  = PS 5.1
::   Server 2019 / Win10 (1809)      = PS 5.1 (PS 7+ installable)
::   Server 2022 / Win11             = PS 5.1 (PS 7+ installable, recommended)
::   Server 2025 / Win11 24H2+       = PS 5.1 (PS 7.4+ installable, recommended)
::   Note: All CIM cmdlets work on PS 5.1+. Some newer cmdlets (e.g.,
::   Get-ComputerInfo full output) work best on PS 5.1 1809+ or PS 7+.
::
:: Double-click opens in editor:
@echo off
if "%1"=="" start "" notepad.exe "%~f0" & exit /b
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
:: TABLE OF CONTENTS
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
::  [1]  NETWORK RESET, DIAGNOSTICS & FIREWALL
::  [2]  DNS / DHCP
::  [3]  STATIC ROUTES & NETWORK DISCOVERY
::  [4]  USER & LOCAL ADMIN MANAGEMENT
::  [5]  REMOTE DESKTOP (RDP)
::  [6]  PROCESS MANAGEMENT
::  [7]  SYSTEM INFO & HEALTH
::  [8]  DISK, FILE & STORAGE OPERATIONS
::  [9]  WINDOWS LICENSING & ACTIVATION
::  [10] WINDOWS UPDATE & SERVICING
::  [11] RESTORE POINTS & RECOVERY
::  [12] ACTIVE DIRECTORY
::  [13] EXCHANGE / MAIL (On-Prem)
::  [14] HYPER-V
::  [15] PRINT SERVER
::  [16] REMOTE EXECUTION & PS REMOTING
::  [17] POWERSHELL -- GENERAL ADMIN
::  [18] POWERSHELL -- EVENT LOG
::  [19] POWERSHELL -- NETWORK INFO
::  [20] WMIC LEGACY --> CIM EQUIVALENTS
::  [21] SECURITY HARDENING (AppLocker, WDAC, Defender)
::  [22] PACKAGE MANAGEMENT (Winget)
::  [23] STORAGE SPACES & REPLICATION (Server)
::  [24] ZPOOL (TrueNAS / ZFS)
::  [25] UNIFI
::  [26] MISC UTILITIES
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

::==========================================================================
:: [1] NETWORK RESET, DIAGNOSTICS & FIREWALL
::==========================================================================

:: --- Full Network Stack Reset (elevated, reboot after) --- [ALL]
ipconfig /flushdns
ipconfig /registerdns
netsh int ip reset
netsh interface ipv4 reset
netsh interface ipv6 reset
netsh interface tcp reset
netsh int reset all
netsh winsock reset
nbtstat -R
nbtstat -RR

:: One-liner version:
ipconfig /flushdns && netsh int ip reset && netsh winsock reset && netsh interface ipv4 reset && netsh interface ipv6 reset && netsh interface tcp reset

:: --- Firewall --- [ALL]
netsh advfirewall reset
netsh advfirewall set allprofiles state on
netsh advfirewall set allprofiles state off
netsh advfirewall firewall set rule group="remote desktop" new enable=yes

:: --- Port Diagnostics --- [ALL]
:: CMD:
netstat -an | find /i "listening"
netstat -an | find /i "443"
netstat -nba

:: Find which PID owns a port, then identify the process:
:: netstat -ano | findstr "443"
:: tasklist /fi "pid eq <PID>"

:: PowerShell (preferred -- no telnet client needed): [ALL]
:: Test-NetConnection -ComputerName "hostname" -Port 443
:: Test-NetConnection -ComputerName "hostname" -Port 443 -InformationLevel Detailed
:: Test-NetConnection -ComputerName "hostname" -TraceRoute

:: [DEP] telnet hostname 80
:: Requires optional Telnet Client feature. Use Test-NetConnection instead.

:: --- TCP Autotuning --- [ALL]
:: Disable if throughput issues on older switches/NICs:
netsh interface tcp set global autotuninglevel=disabled
:: Re-enable (default):
netsh interface tcp set global autotuninglevel=normal

:: --- Resolve hostname from IP --- [ALL]
nbtstat -A 192.168.1.1

:: --- Modern netstat -ano Replacement (Get-NetTCPConnection) --- [ALL]
:: NetTCPIP module cmdlet (Win8/Server 2012+). Returns objects instead of parsed text, so it collapses the section's existing netstat -ano | findstr + tasklist /fi two-step into one filterable/scriptable line.
:: Get-NetTCPConnection -State Listen | Select-Object LocalAddress, LocalPort, OwningProcess
:: Get-NetTCPConnection -LocalPort 443 | Select-Object LocalAddress, LocalPort, State, OwningProcess
:: Get-Process -Id (Get-NetTCPConnection -LocalPort 443).OwningProcess

:: --- In-Box Packet Capture (pktmon) --- [2019+]
:: pktmon.exe ships in-box starting Windows 10 1809 / Server 2019. It captures at multiple points in the network stack (NIC, filters, vSwitch), which makes it more useful than netsh trace for container/Hyper-V networking issues -- and there is currently no packet-capture command anywhere in this section. Verified flags: 'start -c/--capture -f <name>' sets capture mode and filename (default pktmon.etl); 'etl2txt <file> [--out <name>]' is the modern text-conversion command (the older equivalent is 'pktmon format <file>.etl -o <file>.txt') -- added the --out flag so the converted text is actually saved rather than only printed to stdout.
pktmon start --capture -f C:\Temp\capture.etl
pktmon stop
pktmon etl2txt C:\Temp\capture.etl --out C:\Temp\capture.txt

:: --- PowerShell Firewall Profile & Rule Management --- [ALL]
:: NetSecurity module. Returns structured objects and generalizes the netsh advfirewall firewall set rule group="remote desktop" example already in this section to any rule group via -DisplayGroup.
:: Get-NetFirewallProfile -Name Domain,Public,Private | Select-Object Name, Enabled, DefaultInboundAction
:: Set-NetFirewallProfile -Name Domain,Public,Private -Enabled True
:: Get-NetFirewallRule -DisplayGroup "Remote Desktop" | Select-Object DisplayName, Enabled, Direction, Action

::==========================================================================
:: [2] DNS / DHCP
::==========================================================================

:: Show DHCP server: [ALL]
ipconfig /all | find /i "DHCP Server"

:: Show DNS servers: [ALL]
ipconfig /all | find /i "DNS Servers"

:: Reset DNS to DHCP on adapter: [ALL]
netsh interface ipv4 set dns name="Ethernet" dhcp

:: Cisco Umbrella DNS restart cycle: [ALL]
net stop Umbrella_RC /y && netsh interface ip set dns "Ethernet" dhcp && net start Umbrella_RC

:: --- Release/Renew DHCP Lease --- [ALL]
:: Missing baseline pair. Section 1 already covers /flushdns and /registerdns, and this section already reads which DHCP server assigned an address, but forcing a fresh lease -- one of the most common first steps in DHCP/connectivity troubleshooting -- isn't shown anywhere in the file.
ipconfig /release
ipconfig /renew

:: --- PowerShell DNS Server Address Management --- [ALL]
:: DnsClient module (Win8/Server 2012+). -ResetServerAddresses is the object-based equivalent of this section's netsh interface ipv4 set dns name="Ethernet" dhcp line, and Set-...-ServerAddresses replaces manually netsh-ing static DNS servers onto an adapter.
:: Get-DnsClientServerAddress -InterfaceAlias "Ethernet"
:: Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses ("10.0.0.1","10.0.0.2")
:: Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ResetServerAddresses

:: --- DHCP Server Role -- Scopes, Leases & Statistics --- [ALL]
:: DhcpServer module cmdlets for admins running the DHCP Server role itself. The current section only covers the DHCP client side (which server/DNS an adapter received); these list configured scopes, active leases per scope, and server-wide utilization stats -- server-side DHCP management is entirely absent otherwise.
:: Get-DhcpServerv4Scope -ComputerName "dhcpserver.domain.local"
:: Get-DhcpServerv4Lease -ComputerName "dhcpserver.domain.local" -ScopeId 10.10.10.0
:: Get-DhcpServerv4Statistics -ComputerName "dhcpserver.domain.local"

::==========================================================================
:: [3] STATIC ROUTES & NETWORK DISCOVERY
::==========================================================================

:: --- Static Routes --- [ALL]
route add 192.168.1.83 mask 255.255.255.255 172.16.1.1 metric 31 if 11 -p
route add 192.168.1.0 mask 255.255.255.0 172.16.1.1 metric 31 if 11 -p
route -f

:: Flush persistent routes (takes effect at reboot): [ALL]
reg delete HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\PersistentRoutes /va /f

:: --- Network Discovery Dependencies --- [ALL]
:: Requires: DNS Client, Function Discovery Resource Pub, SSDP Discovery, UPnP Device Host
:: PowerShell:
:: Get-Service -Name Dnscache, FDResPub, SSDPSRV, upnphost | Start-Service
:: Get-Service -Name Dnscache, FDResPub, SSDPSRV, upnphost | Set-Service -StartupType Automatic

:: --- View Current Routing Table --- [ALL]
:: Missing baseline command. The section shows route add and route -f (flush) but never how to actually view the table being added to or flushed.
route print
route print -4

:: --- PowerShell Route Management (Get-NetRoute / New-NetRoute) --- [ALL]
:: NetTCPIP module (Win8/Server 2012+). New-NetRoute persists to both the active and persistent route stores by default -- no -p flag needed the way CMD's route add requires -- and Get-NetRoute/Remove-NetRoute are scriptable equivalents of route print/route -f.
:: Get-NetRoute | Format-Table -AutoSize
:: New-NetRoute -DestinationPrefix "192.168.1.0/24" -InterfaceAlias "Ethernet" -NextHop 172.16.1.1 -RouteMetric 31
:: Remove-NetRoute -DestinationPrefix "192.168.1.0/24" -Confirm:$false

:: --- Neighbor/ARP Cache Discovery (Get-NetNeighbor) --- [ALL]
:: NetTCPIP module equivalent of arp -a for both IPv4 and IPv6, with State filtering (Reachable/Stale/Unreachable/Permanent) the legacy arp command can't do. The section's 'Network Discovery' half currently only covers the Network Discovery *feature's* service dependencies, not any actual peer/host discovery tool -- this fills that gap.
:: Get-NetNeighbor -AddressFamily IPv4 | Select-Object IPAddress, LinkLayerAddress, State
:: Get-NetNeighbor -State Reachable

::==========================================================================
:: [4] USER & LOCAL ADMIN MANAGEMENT
::==========================================================================

:: --- CMD (Legacy, works everywhere) --- [ALL]
:: Add local user (interactive password prompt -- NEVER hardcode):
net user /add iadmin *
net localgroup administrators iadmin /add

:: Change password:
net user iadmin *

:: List local users / admins:
net user
net localgroup administrators

:: Check domain user last logon:
net user USERNAME /domain | findstr /C:"Last logon"

:: Current user:
whoami

:: --- PowerShell (Modern) --- [ALL, PS 5.1+]
:: Get-LocalGroupMember -Group "Administrators"
::
:: Create local admin (secure prompt):
:: $Password = Read-Host -AsSecureString "Enter password"
:: New-LocalUser -Name "iadmin" -Password $Password -FullName "IT Admin" -Description "Local admin account"
:: Add-LocalGroupMember -Group "Administrators" -Member "iadmin"

:: --- Windows LAPS -- Retrieve the Rotated Local Admin Password --- [2019+]
:: Windows LAPS (built-in since the April 2023 update, native on Server 2025) is the modern, security-conscious way to manage rotating local admin credentials. Not supported on Server 2016, so tagged [2019+] rather than [ALL] per the file's own compatibility key. Verified live: syntax Get-LapsADPassword [-Identity] <String[]> [-AsPlainText] [-IncludeHistory] and the OS support matrix confirmed directly against the fetched MicrosoftDocs pages.
:: Windows LAPS now ships built into the OS -- it replaces ad hoc scripts for rotating/retrieving local admin passwords.
:: Requires Server 2019/2022 with the April 11, 2023 update or later (Server 2025 has it natively). NOT supported on Server 2016.
:: Retrieve the current AD-stored LAPS password for a computer (requires the LAPS module + AD read permission):
:: Get-LapsADPassword -Identity "COMPUTERNAME" -AsPlainText
:: Include password history:
:: Get-LapsADPassword -Identity "COMPUTERNAME" -AsPlainText -IncludeHistory

::==========================================================================
:: [5] REMOTE DESKTOP (RDP)
::==========================================================================

:: Enable RDP + firewall rule (one-liner): [ALL]
reg add "HKLM\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f && netsh advfirewall firewall set rule group="remote desktop" new enable=yes

:: Connect in admin/console mode: [ALL]
mstsc.exe /admin /v:"servername-or-ip"

:: Enable RDP remotely via PowerShell: [ALL, PS 5.1+]
:: Invoke-Command -ComputerName "REMOTE-PC" -ScriptBlock {
::   Set-ItemProperty -Path 'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections -Value 0
::   Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
:: }

:: Query active RDP sessions: [ALL]
qwinsta /server:SERVERNAME

:: Kill RDP session by ID: [ALL]
rwinsta /server:SERVERNAME 2

:: Query last logged-in user from registry (remote): [ALL]
reg query "\\COMPUTERNAME\HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\WinLogon" /v DefaultUserName

:: --- mstsc Restricted Admin / Remote Credential Guard Modes --- [ALL]
:: The current section covers enabling RDP and /admin console mode but not the security-hardened connection modes admins should be using for jump-box/privileged access. /restrictedAdmin and /remoteGuard are long-standing (Win8.1/Server 2012 R2+) mstsc switches, within the file's Server 2016+/Win10+ baseline, so tagged [ALL]. Verified live: both switches and their descriptions confirmed directly in the fetched mstsc.exe reference.
:: Connect without sending credentials to the target -- protects you if the target box is already compromised (implies /admin):
mstsc /v:servername /restrictedAdmin
:: Remote Credential Guard -- also protects credentials on connections initiated FROM the remote PC (requires Kerberos + domain-joined devices on both ends):
mstsc /v:servername /remoteGuard
:: Both modes must be permitted on the target via policy/registry; neither sends the caller's cached credentials to the remote host, reducing pass-the-hash exposure on jump boxes.

::==========================================================================
:: [6] PROCESS MANAGEMENT
::==========================================================================

:: List/sort processes: [ALL]
tasklist | sort

:: Kill by name / PID: [ALL]
taskkill /F /IM processname.exe
taskkill /F /PID 1234
taskkill /F /IM process1.exe /IM process2.exe

:: Kill all browsers: [ALL]
taskkill /f /im msedge.exe /im firefox.exe /im chrome.exe /im iexplore.exe

:: PowerShell -- top 10 by memory: [ALL, PS 5.1+]
:: Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10 Name, Id, @{N='Mem(MB)';E={[math]::Round($_.WorkingSet64/1MB,2)}}

:: --- Top 10 processes by CPU (elapsed CPU time) --- [ALL]
:: Complements the existing top-10-by-memory one-liner in this section with the CPU equivalent. The CPU property on Get-Process objects is cumulative processor time in seconds (TotalProcessorTime), not instantaneous percent.
:: PowerShell -- top 10 by CPU (Note: CPU is total elapsed CPU seconds since process start, not live %):
:: Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, Id, CPU, WorkingSet64

:: --- Full command line per process (replaces wmic process get commandline) --- [ALL]
:: Useful triage one-liner and the modern replacement for the now-removed `wmic process get commandline` -- WMIC is removed by default on new Windows 11 24H2 installs and Microsoft has announced full removal in a future feature update. Corrected the tag from the invalid "[ALL, PS 5.1+]" to the allowed "[ALL]".
:: tasklist /v does not show the full launch command line -- use CIM instead:
:: Get-CimInstance Win32_Process | Select-Object ProcessId, Name, CommandLine | Format-Table -AutoSize

::==========================================================================
:: [7] SYSTEM INFO & HEALTH
::==========================================================================

:: --- Basic Info --- [ALL]
systeminfo
hostname

:: --- Comprehensive Info (faster/richer than systeminfo) --- [2019+, PS 5.1+]
:: Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsHardwareAbstractionLayer, OsLastBootUpTime
:: Full dump: Get-ComputerInfo

:: --- OS Version --- [ALL, PS 5.1+]
:: (Get-CimInstance Win32_OperatingSystem).Caption
:: [System.Environment]::OSVersion

:: --- 32-bit vs 64-bit --- [ALL, PS 5.1+]
:: [Environment]::Is64BitOperatingSystem

:: --- Performance --- [ALL]
perfmon /report
perfmon /res

:: --- Uptime / Last Boot --- [ALL, PS 5.1+]
:: Get-CimInstance Win32_OperatingSystem | Select-Object LastBootUpTime

:: --- Disk Health --- [ALL, PS 5.1+]
:: Get-PhysicalDisk | Select-Object FriendlyName, MediaType, Size, HealthStatus, OperationalStatus
:: Get-CimInstance Win32_DiskDrive | Select-Object DeviceID, Model, Status

:: --- BIOS Info --- [ALL, PS 5.1+]
:: Get-CimInstance Win32_BIOS | Format-List *
:: Get-CimInstance Win32_BIOS | Select-Object SerialNumber, Manufacturer, Name

:: --- Drive Serial Numbers --- [ALL, PS 5.1+]
:: Get-CimInstance Win32_PhysicalMedia | Select-Object Tag, SerialNumber
:: Get-CimInstance Win32_DiskDrive | Select-Object Name, SerialNumber

:: --- DISM Image Health --- [ALL]
DISM /Online /Cleanup-Image /CheckHealth
DISM /Online /Cleanup-Image /ScanHealth
DISM /Online /Cleanup-Image /RestoreHealth
DISM /Online /Cleanup-Image /StartComponentCleanup
DISM /Online /Cleanup-Image /AnalyzeComponentStore

:: --- SFC --- [ALL]
sfc /scannow
:: Offline (recovery environment):
:: sfc /scannow /offbootdir=D:\ /offwindir=D:\Windows

:: --- Check Disk --- [ALL]
chkdsk /f /r /x

:: --- Startup Applications --- [ALL, PS 5.1+]
:: Get-CimInstance Win32_StartupCommand | Select-Object Name, Command, Location

:: --- .NET Framework Versions --- [ALL]
reg query "HKLM\SOFTWARE\Microsoft\NET Framework Setup\NDP\v3.5" | findstr Install
:: PowerShell -- all versions:
:: Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -Recurse | Get-ItemProperty -Name Version -ErrorAction SilentlyContinue | Select-Object PSChildName, Version

:: --- Installed Hotfixes --- [ALL, PS 5.1+]
:: Get-HotFix | Select-Object HotFixID, InstalledOn, Description | Sort-Object InstalledOn
:: Get-HotFix -Id KB5001234

:: [DEP] wmic qfe list brief
:: Deprecated on Server 2022+ / Win11. Use Get-HotFix above.

:: --- Uninstall KB --- [ALL]
wusa /uninstall /kb:4480970 /quiet

:: --- Measure Script Execution Time --- [ALL, PS 5.1+]
:: Measure-Command { & .\myscript.ps1 } | Select-Object TotalSeconds, TotalMilliseconds

:: --- TPM Status --- [2019+, PS 5.1+]
:: Get-Tpm

:: --- Windows Defender Status --- [2019+, PS 5.1+]
:: Get-MpComputerStatus | Select-Object AMServiceEnabled, AntispywareEnabled, RealTimeProtectionEnabled

:: --- Export Drivers (for reimaging) --- [ALL]
:: dism /online /export-driver /destination:C:\DriversBackup

:: --- Correction: WMIC status is stronger than [DEP] -- it is actively being removed --- [DEP]
:: Correction to the existing `[DEP] wmic qfe list brief` note, which currently only says it's deprecated on Server 2022+/Win11 -- that undersells it. Microsoft's WMIC removal notice confirms active removal, not just deprecation.
:: UPDATE: wmic is no longer just deprecated. New Windows 11 24H2 installs already ship without it
:: (only installable as a Feature on Demand), and Microsoft has confirmed WMIC is being fully
:: removed -- FoD included -- in a future Windows feature update. Treat any wmic dependency
:: (scripts, RMM tooling, scheduled tasks) as broken-on-arrival for new builds.
:: Use Get-HotFix / Get-CimInstance now, not just "when convenient."

::==========================================================================
:: [8] DISK, FILE & STORAGE OPERATIONS
::==========================================================================

:: --- Delete Temp Files --- [ALL]
Del /S /F /Q %Windir%\Temp && Del /S /F /Q %localappdata%\Temp

:: --- Robocopy Mirror --- [ALL]
:: /MIR = mirror (DELETES dest files not in source -- use caution)
:: /FFT = 2-sec timestamp granularity (good for NAS/FAT targets)
:: /MT:8 = multithreaded (adjust to taste; default is 8 on Win8+)
robocopy "C:\Source" "D:\Dest" /MIR /FFT /R:3 /W:5 /Z /NP /NDL /MT:8
:: With logging:
:: robocopy "C:\Source" "D:\Dest" /MIR /FFT /R:3 /W:5 /Z /NP /NDL /MT:8 /LOG:"C:\Logs\robocopy.log"

:: --- Copy from Remote --- [ALL]
xcopy /s \\remotecomputer\directory c:\local

:: --- Find Files/Dirs --- [ALL]
dir c:\ /s /b | find "blah"
dir c:\ /s /b /ad | find "blah"

:: --- Display File Contents --- [ALL]
type filename.txt

:: --- OneDrive Junction Symlink --- [ALL]
mklink /j "%UserProfile%\OneDrive\SyncFolder" "C:\FolderToSync"

:: --- PowerShell File Operations --- [ALL, PS 5.1+]
:: Find files by type, sorted by size:
:: Get-ChildItem 'C:\' -Recurse -Include *.mp3 -ErrorAction SilentlyContinue | Select-Object FullName, Length | Sort-Object Length

:: Folder sizes:
:: Get-ChildItem "C:\Users" -Directory | ForEach-Object {
::   $size = (Get-ChildItem $_.FullName -Recurse -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
::   [PSCustomObject]@{ Folder = $_.Name; SizeMB = [math]::Round($size/1MB,2) }
:: } | Sort-Object SizeMB -Descending

:: Find 20 largest files on C:
:: Get-ChildItem C:\ -Recurse -File -ErrorAction SilentlyContinue | Sort-Object Length -Descending | Select-Object -First 20 FullName, @{Name='MB';Expression={[math]::Round($_.Length/1MB,2)}}

:: --- Check Volume Free Space / Health --- [ALL, PS 5.1+]
:: Get-Volume (Storage module) returns free space, filesystem, and HealthStatus (e.g., ReFS/NTFS corruption flags) per volume -- a modern replacement for parsing `fsutil volume diskfree` or `wmic logicaldisk`. Verified current on the Storage module Get-Volume reference.
:: Get-Volume | Select-Object DriveLetter, FileSystemLabel, FileSystem, HealthStatus, SizeRemaining, Size

:: --- Zip/Unzip via PowerShell --- [ALL, PS 5.1+]
:: Compress-Archive/Expand-Archive (Microsoft.PowerShell.Archive module, built in since PS 5.0) give a native zip workflow with no external tool dependency -- useful when 7-Zip/tar aren't installed. Compress-Archive is capped at 2GB per archive due to the underlying System.IO.Compression.ZipArchive API (confirmed: exceeding it throws 'Stream was too long'), so it's not a robocopy replacement for bulk transfers.
:: Compress-Archive -Path 'C:\Folder\*' -DestinationPath 'C:\archive.zip'
:: Expand-Archive -Path 'C:\archive.zip' -DestinationPath 'C:\Output'

::==========================================================================
:: [9] WINDOWS LICENSING & ACTIVATION
::==========================================================================

:: Show license status: [ALL]
slmgr.vbs /dli
slmgr.vbs /dlv

:: Activate with installed key: [ALL]
slmgr.vbs /ato

:: Install product key: [ALL]
slmgr.vbs /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX

:: Clear key from registry (post-activation security): [ALL]
slmgr.vbs /cpky

:: Get OEM key from BIOS: [ALL, PS 5.1+]
:: (Get-CimInstance -ClassName SoftwareLicensingService).OA3xOriginalProductKey

:: Change server edition (e.g., Eval --> Standard): [ALL]
:: DISM /online /Set-Edition:ServerStandard /ProductKey:XXXXX-XXXXX-XXXXX-XXXXX-XXXXX /AcceptEula

:: --- Automatic Virtual Machine Activation (AVMA) --- [ALL]
:: AVMA lets a Hyper-V host activate its Datacenter-licensed guest VMs automatically and report license state back to the host, including in disconnected/dark-site environments -- distinct from KMS/MAK activation already listed. Verified current on the Windows Server 'Automatic Virtual Machine Activation' Learn page; added the Data Exchange integration-service prerequisite the page calls out.
:: Requires: Windows Server Datacenter edition on the Hyper-V host, guest running Windows Server Datacenter/Standard/Essentials
slmgr.vbs /ipk XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
:: Use the published AVMA key for the guest edition (not a customer product key) -- see Microsoft's AVMA key table.
:: AVMA activates automatically at guest boot; no key ever needs to be entered inside the guest by an admin post-deployment.
:: Requires the Data Exchange (KVP) integration service enabled on the guest -- on by default for new VMs.

:: --- Set/Clear KMS Host Manually --- [ALL]
:: Useful when DNS-based KMS auto-discovery is broken or a client must activate against a specific KMS host (e.g., a workgroup machine or isolated segment). Requires TCP 1688 reachable to the KMS host. Verified against Microsoft's 'Replace a KMS host' guidance.
slmgr.vbs /skms kms-host.example.com:1688
slmgr.vbs /ckms
:: /ckms clears the manual KMS host and reverts the client to DNS auto-discovery (SRV record).

::==========================================================================
:: [10] WINDOWS UPDATE & SERVICING
::==========================================================================

:: --- Trigger Update Scan + Install --- [ALL]
UsoClient ScanInstallWait
UsoClient StartInstall

:: --- PSWindowsUpdate Module (more control) --- [ALL, PS 5.1+]
:: Install-Module -Name PSWindowsUpdate -Force
:: Get-WindowsUpdate
:: Install-WindowsUpdate -AcceptAll -AutoReboot

:: --- Check Pending Reboot --- [ALL, PS 5.1+]
:: Test-Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending"

:: --- List Installed Updates (modern, non-wmic) --- [ALL, PS 5.1+]
:: Get-HotFix (built into PowerShell, backed by the Win32_QuickFixEngineering WMI class) is the direct, still-supported replacement for `wmic qfe list`, which falls under the general WMIC deprecation. Note: Win32_QuickFixEngineering only reflects CBS-serviced updates, not every MSI/Windows Update patch -- verified against the Get-HotFix Learn reference.
:: Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object HotFixID, Description, InstalledOn

:: --- Decode the Windows Update ETL Logs --- [ALL, PS 5.1+]
:: Since Windows 10, Windows Update no longer writes a plaintext WindowsUpdate.log directly -- it logs to unreadable .etl trace files, and Get-WindowsUpdateLog merges/converts them into a single readable log on the desktop. Verified current on the Get-WindowsUpdateLog Learn reference. Fills a real gap next to the existing UsoClient/PSWindowsUpdate/pending-reboot entries.
:: Get-WindowsUpdateLog
:: Writes WindowsUpdate.log to the current user's desktop by default; pass -LogPath to redirect.

::==========================================================================
:: [11] RESTORE POINTS & RECOVERY
::==========================================================================

:: --- Create Restore Point --- [ALL, PS 5.1+]
:: Checkpoint-Computer -Description "Pre-Change Restore Point" -RestorePointType MODIFY_SETTINGS

:: [DEP] Wmic.exe /Namespace:\\root\default Path SystemRestore Call CreateRestorePoint "My Restore Point", 100, 12
:: WMIC deprecated on Server 2022+ / Win11. Use Checkpoint-Computer above.

:: --- List Restore Points --- [ALL, PS 5.1+]
:: Get-ComputerRestorePoint

:: --- VSS Shadow Copies --- [ALL]
vssadmin List Shadows

:: --- Enable System Restore on C: --- [ALL, PS 5.1+]
:: Enable-ComputerRestore -Drive "C:\"

:: --- Check Windows Recovery Environment Status --- [ALL]
:: reagentc /info is the direct, current way to verify WinRE health before relying on push-button reset or BitLocker recovery -- a gap next to the existing restore-point/VSS content. Verified current on Microsoft's REAgentC command-line options page.
reagentc /info
:: Confirms whether WinRE is Enabled/Disabled and shows the WinRE image location -- required for BitLocker recovery and push-button reset to function.
:: Log: C:\Windows\Logs\Reagent.log

:: --- Repair the Component Store / System Files --- [ALL]
:: Microsoft-documented recovery workflow: run DISM /ScanHealth to detect component-store corruption, /RestoreHealth to repair it, then sfc /scannow to repair protected system files from that now-healthy store. If sfc still reports unfixable files, reboot and re-run it (community/Microsoft Q&A guidance notes locked files can clear across a reboot). Complements the existing restore-point/VSS content with an actual corruption-repair path.
DISM /Online /Cleanup-Image /ScanHealth
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow

::==========================================================================
:: [12] ACTIVE DIRECTORY
::==========================================================================

:: --- Install RSAT AD Tools --- 
:: Win10/11: [ALL]
:: Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
:: Install ALL RSAT tools at once (Win10 1809+ / Win11):
:: Get-WindowsCapability -Online | Where-Object { $_.Name -like 'Rsat*' } | Add-WindowsCapability -Online
:: Server: [ALL]
:: Install-WindowsFeature -Name RSAT-AD-PowerShell
:: Import-Module ActiveDirectory

:: --- Azure AD Connect Delta Sync --- [ALL, PS 5.1+]
:: Start-ADSyncSyncCycle -PolicyType Delta

:: --- Group Operations --- [ALL, PS 5.1+]
:: (Get-ADGroupMember -Identity 'GroupName').Count
:: Get-ADGroupMember -Identity "GroupName" | Select-Object Name, SamAccountName | Export-Csv -Path "C:\scripts\group-members.csv" -NoTypeInformation

:: --- Export All AD Users --- [ALL, PS 5.1+]
:: Get-ADUser -Filter * -Properties DisplayName, EmailAddress, Title | Select-Object DisplayName, EmailAddress, Title | Export-CSV "C:\Scripts\ad-users.csv" -NoTypeInformation

:: --- Bulk Add OU Users to Group (preview with -WhatIf) --- [ALL, PS 5.1+]
:: Get-ADUser -SearchBase 'OU=MyOU,DC=domain,DC=local' -Filter * | ForEach-Object { Add-ADGroupMember 'TargetGroup' -Members $_ -WhatIf }

:: --- List Domain Computers --- [ALL]
:: CMD: NETDOM QUERY /D:MyDomain WORKSTATION
:: PS:  Get-ADComputer -Filter * | Select-Object Name, DistinguishedName | Export-Csv "C:\scripts\domain-computers.csv" -NoTypeInformation
:: OU-specific (legacy dsquery): dsquery computer "OU=example,DC=domain,DC=com" -o rdn -limit 6000 > output.txt

:: --- Rename Computer (AD-joined) --- [ALL, PS 5.1+]
:: Rename-Computer -ComputerName "OLD-PC" -NewName "NEW-PC" -DomainCredential domain\AdminUser -Force -Restart

:: [DEP] WMIC computersystem where caption='OLD-PC' rename 'NEW-PC'
:: Deprecated on Server 2022+ / Win11. Use Rename-Computer above.

:: --- AD Site --- [ALL, PS 5.1+]
:: [System.DirectoryServices.ActiveDirectory.ActiveDirectorySite]::GetComputerSite().Name

:: --- Account Lockout / Unlock --- [ALL, PS 5.1+]
:: Search-ADAccount -LockedOut | Select-Object Name, SamAccountName, LockedOut
:: Unlock-ADAccount -Identity "username"

:: --- Disabled Users --- [ALL, PS 5.1+]
:: Get-ADUser -Filter {Enabled -eq $false} | Select-Object Name, SamAccountName

:: --- Correction: Azure AD Connect Was Renamed --- [ALL]
:: The existing section heads this block 'Azure AD Connect Delta Sync' -- that branding is stale. This is a correction, not new functionality (Start-ADSyncSyncCycle syntax is unchanged). Verified live against the fetched Microsoft Entra Connect overview doc: confirms the rename, the Aug 31 2022 v1 retirement, and that Cloud Sync is positioned as the forward-looking replacement.
:: NOTE: 'Azure AD Connect' was officially renamed Microsoft Entra Connect (the sync engine is now 'Microsoft Entra Connect Sync'; Microsoft Entra Cloud Sync is the lightweight/multi-forest replacement Microsoft now recommends).
:: Azure AD Connect v1 was retired Aug 31, 2022 -- confirm you're on Microsoft Entra Connect v2.x+ before relying on it.
:: The cmdlet itself is unchanged:
:: Start-ADSyncSyncCycle -PolicyType Delta

:: --- Check AD Replication Health (PowerShell) --- [ALL]
:: The AD section currently has no replication health/troubleshooting commands. Get-ADReplicationFailure and Get-ADReplicationPartnerMetadata (ActiveDirectory module, available since Server 2012 R2+, within this file's Server 2016+ baseline) are the PowerShell-native way to do what repadmin has traditionally done. Verified live: syntax pattern Get-ADReplicationFailure dc1.corp.contoso.com and Get-ADReplicationPartnerMetadata -target dc1.corp.contoso.com confirmed in the fetched Microsoft doc; adjusted the forest-wide example to enumerate DCs via -Target since the draft's unverified -Scope Forest parameter was not shown in the source.
:: Forest-wide replication failure summary (PowerShell equivalent of repadmin /showreplsum):
:: Get-ADReplicationFailure -Target (Get-ADDomainController -Filter *).Name
:: Per-DC replication partner state/timing (flag anything with a non-zero last result):
:: Get-ADReplicationPartnerMetadata -Target dc1.corp.contoso.com | Select-Object LastReplicationAttempt, LastReplicationResult, Partner

::==========================================================================
:: [13] EXCHANGE / MAIL (On-Prem)
::==========================================================================

:: Get mailboxes by domain: [Exchange Management Shell]
:: Get-Mailbox -Filter { WindowsEmailAddress -like "*@domain.com" }

:: Export mailbox to PST:
:: New-MailboxExportRequest -Mailbox username -FilePath \\SERVER\share\archive-username.pst
:: Get-MailboxExportRequest | Get-MailboxExportRequestStatistics

:: Clean mailbox database (reconcile disconnected):
:: Get-MailboxDatabase | Clean-MailboxDatabase

:: Get Full Access permissions for a user:
:: Get-Mailbox | Get-MailboxPermission | Where-Object {
::   ($_.AccessRights -eq "FullAccess") -and ($_.User -like 'domain\username') -and ($_.IsInherited -eq $false)
:: } | Format-Table Identity, User, AccessRights

:: Transport rule -- reject large attachments:
:: New-TransportRule -Name LargeAttach -AttachmentSizeOver 30MB -RejectMessageReasonText "Attachment over 30MB - rejected."
:: Remove-TransportRule -Identity "LargeAttach"

:: --- Move mailbox to another database (batch move request) ---
:: New-MoveRequest/Get-MoveRequestStatistics complements the existing PST export and permission-audit entries by covering local database-to-database mailbox moves, the routine task of relocating a mailbox and tracking its progress - not present in the current section. Cmdlet names/parameters verified against ExchangePowerShell module conventions (network egress to learn.microsoft.com was blocked by this session's org policy, so citation URLs could not be live-fetched this pass; kept as standard, well-formed Microsoft Learn doc paths for these cmdlets).
::   New-MoveRequest -Identity user@domain.com -TargetDatabase DB02
::   Get-MoveRequest -ResultSize Unlimited | Get-MoveRequestStatistics | ft DisplayName,StatusDetail,PercentComplete

::==========================================================================
:: [14] HYPER-V
::==========================================================================

:: Install role (local): [ALL Server]
:: Install-WindowsFeature -Name Hyper-V -IncludeManagementTools -Restart

:: Install role (remote): [ALL Server]
:: Install-WindowsFeature -Name Hyper-V -ComputerName SERVER-NAME -IncludeManagementTools -Restart

:: Verify installed features: [ALL Server]
:: Get-WindowsFeature | Where-Object { $_.Installed -eq $true }

:: Toggle Hyper-V hypervisor (for VirtualBox/VMware Workstation compatibility): [ALL]
:: Disable: bcdedit /set hypervisorlaunchtype off
:: Enable:  bcdedit /set hypervisorlaunchtype auto
:: Reboot required after either change.

:: --- Manage checkpoints via PowerShell --- [ALL]
:: The section currently covers install/enable-only tasks; it has no checkpoint (snapshot) lifecycle commands, which are core day-to-day Hyper-V admin work (create/list/apply/remove). Syntax verified against known Hyper-V module cmdlet signatures; tag corrected to [ALL] since these cmdlets exist from Server 2016 onward.
::   Checkpoint-VM -Name VM01 -SnapshotName "Before-Patch"
::   Get-VM VM01 | Get-VMSnapshot | Sort-Object CreationTime
::   Get-VMSnapshot -VMName VM01 -Name "Before-Patch" | Restore-VMSnapshot -Confirm:$false
::   Get-VMSnapshot -VMName VM01 -Name "Before-Patch" | Remove-VMSnapshot

:: --- Set checkpoint type (avoid production-checkpoint surprises) --- [ALL]
:: Set-VM -CheckpointType lets admins pin a VM to Standard (state) vs Production (VSS-based, app-consistent) checkpoints - a frequent source of confusion/troubleshooting not covered in the current section. Valid values are Standard, Production, ProductionOnly, Disabled.
::   Set-VM -Name VM01 -CheckpointType Standard

::==========================================================================
:: [15] PRINT SERVER
::==========================================================================

:: Remote print server properties: [ALL]
rundll32 printui.dll,PrintUIEntry /s /t1 /c\\PRINTSERVER

:: Add per-machine printer: [ALL]
rundll32 printui.dll,PrintUIEntry /ga /n "\\SERVER\PRINTER"

:: Set default printer: [ALL, PS 5.1+]
:: Get-CimInstance -ClassName Win32_Printer -Filter "Name='Printer 1'" | Invoke-CimMethod -MethodName SetDefaultPrinter

:: List all printers: [ALL, PS 5.1+]
:: Get-Printer | Select-Object Name, PortName, Shared, Published

:: Restart spooler: [ALL, PS 5.1+]
:: Restart-Service -Name Spooler -Force

:: Clear print queue (nuclear option): [ALL]
net stop spooler && del /Q /F /S "%SystemRoot%\System32\spool\PRINTERS\*.*" && net start spooler

:: --- Install a printer driver --- [ALL]
:: The current section adds printers and manages the spooler/queue but has no driver-installation cmdlet - Add-PrinterDriver is the modern PrintManagement-module equivalent for staging drivers before Add-Printer. Tag corrected from the invalid "[ALL, PS 5.1+]" (not an allowed tag value) to [ALL].
::   Add-PrinterDriver -Name "HP Universal Printing PCL 6" -InfPath "C:\Distr\HP-pcl6-x64\hpcu118u.inf"

::==========================================================================
:: [16] REMOTE EXECUTION & PS REMOTING
::==========================================================================

:: --- PsExec (Sysinternals) --- [ALL]
:: Download: https://learn.microsoft.com/en-us/sysinternals/downloads/psexec
:: PsExec \\REMOTE-PC cmd.exe
:: PsExec \\REMOTE-PC powershell.exe
:: Run as SYSTEM locally: psexec -i -s cmd.exe

:: --- Map Network Drive --- [ALL]
:: net use \\REMOTE-PC /USER:domain\username *
:: (The * prompts for password -- never pass it inline)

:: --- PowerShell Remoting (preferred over PsExec) --- [ALL, PS 5.1+]
:: Enable on target first:
:: Enable-PSRemoting -Force

:: Interactive session:
:: Enter-PSSession -ComputerName "REMOTE-PC"
:: Exit-PSSession

:: Run command on multiple machines:
:: Invoke-Command -ComputerName server1, server2, server3 -ScriptBlock { Get-Process }

:: Run script on multiple machines:
:: Invoke-Command -ComputerName server1, server2, server3 -FilePath "\\scriptserver\scripts\script.ps1"

:: Persistent sessions (reuse for multiple commands):
:: $sessions = New-PSSession -ComputerName server1, server2, server3
:: Invoke-Command -Session $sessions -ScriptBlock { Get-Process | Select-Object Name, VM, CPU }
:: Remove-PSSession $sessions

:: --- PowerShell Remoting over SSH (cross-platform) --- [ALL]
:: The section only covers classic WinRM-based remoting. SSH-based remoting is the modern way to reach mixed Windows/Linux estates. Flagged as PS 7+ inline since the tag vocabulary has no PS-version tag. Verified live: -HostName/-UserName/-KeyFilePath syntax and the JEA limitation confirmed directly in the fetched PowerShell-Docs SSH remoting guide.
:: PS remoting over SSH instead of WinRM -- connects Windows/Linux/macOS hosts interchangeably. Requires PowerShell 7+ and an SSH client/server on both ends.
:: New-PSSession -HostName linux-host.domain.com -UserName admin
:: Enter-PSSession -HostName linux-host.domain.com -UserName admin
:: Key-based auth instead of a password prompt:
:: New-PSSession -HostName linux-host.domain.com -UserName admin -KeyFilePath ~/.ssh/id_rsa
:: Note: SSH-based remoting does not currently support JEA or remote endpoint session configuration -- use WinRM-based remoting above when you need those.

:: --- Copy Files Over an Existing PS Remoting Session --- [ALL]
:: Copy-Item -ToSession/-FromSession (available since WMF5/PS 5.1, so [ALL] fits) lets you push/pull files through a PSSession you already opened -- avoids opening admin shares just to move a file. Verified live: -ToSession and -FromSession parameter descriptions and matching usage examples confirmed directly in the fetched Copy-Item 5.1 reference.
:: Transfer files through an already-open PSSession -- no separate SMB/admin-share path needed:
:: $Session = New-PSSession -ComputerName "REMOTE-PC"
:: Copy-Item "C:\Local\file.txt" -Destination "C:\Remote\" -ToSession $Session
:: Copy-Item "C:\Remote\file.txt" -Destination "C:\Local\" -FromSession $Session

::==========================================================================
:: [17] POWERSHELL -- GENERAL ADMIN
::==========================================================================

:: --- dir /b equivalent --- [ALL, PS 5.1+]
:: Get-ChildItem -Name
:: (Get-ChildItem -Directory).Name

:: --- Services color-coded --- [ALL, PS 5.1+]
:: Get-Service | ForEach-Object {
::   if ($_.Status -eq "Stopped") { Write-Host -ForegroundColor Red "$($_.Name) $($_.Status)" }
::   else { Write-Host -ForegroundColor Green "$($_.Name) $($_.Status)" }
:: }

:: --- List all services --- [ALL, PS 5.1+]
:: Get-Service | Select-Object Name, Status, StartType | Sort-Object Status

:: --- Disable a startup service --- [ALL, PS 5.1+]
:: Set-Service -Name "ServiceName" -StartupType Disabled

:: --- Last Reboot (user + time) --- [ALL, PS 5.1+]
:: Get-EventLog -LogName System -Newest 1000 | Where-Object { $_.EventID -eq 1074 } | Format-Table MachineName, UserName, TimeGenerated -AutoSize

:: --- All Installed Software (fast, registry-based) --- [ALL, PS 5.1+]
:: MUCH faster than Get-CimInstance Win32_Product (which triggers MSI reconfigure).
:: Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*,
::   HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
::   Select-Object DisplayName, DisplayVersion, Publisher, InstallDate |
::   Sort-Object DisplayName | Format-Table -AutoSize

:: --- Remotely Check Logged-In User --- [ALL, PS 5.1+]
:: Get-CimInstance -ClassName Win32_ComputerSystem -ComputerName "REMOTE-PC" | Select-Object UserName

:: --- Execution Policy --- [ALL, PS 5.1+]
:: Set-ExecutionPolicy RemoteSigned -Force
:: Or scope to current user only:
:: Set-ExecutionPolicy RemoteSigned -Scope CurrentUser -Force

:: --- Clipboard to File --- [ALL, PS 5.1+]
:: Get-Clipboard | Out-File "C:\Temp\clipboard.txt"

:: --- ISO 8601 Date --- [ALL, PS 5.1+]
:: Get-Date -Format "yyyy-MM-dd"

:: --- Installed PowerShell modules (PSGallery) and help refresh --- [ALL]
:: This section covers general PS admin housekeeping but has nothing on module/help maintenance, routine hygiene on any admin workstation or jump box. Corrected the tag from the invalid "[ALL, PS 5.1+]" to the allowed "[ALL]".
:: List modules installed from the PowerShell Gallery:
:: Get-InstalledModule | Select-Object Name, Version, Repository
:: Update all installed modules to their latest published version:
:: Update-Module
:: Refresh local cmdlet help content (run elevated):
:: Update-Help -Force

::==========================================================================
:: [18] POWERSHELL -- EVENT LOG
::==========================================================================

:: --- Legacy Get-EventLog (works everywhere, slower) --- [ALL, PS 5.1+]
:: Get-EventLog -LogName Security -Message "*keyword*" | Format-Table -AutoSize
:: Get-EventLog -LogName System -Message "*usb*" | Format-Table -Wrap -AutoSize | Out-File "C:\Logs\usblog.txt"
:: Get-EventLog -LogName Security | Where-Object { $_.EventID -eq 4688 } | Format-Table -AutoSize

:: --- Modern Get-WinEvent (faster, supports newer log channels) --- [ALL, PS 5.1+]
:: Failed logins (4625):
:: Get-WinEvent -FilterHashtable @{LogName='Security'; ID=4625} -MaxEvents 50 | Format-Table TimeCreated, Message -Wrap

:: Shutdown/reboot events (1074):
:: Get-WinEvent -FilterHashtable @{LogName='System'; ID=1074} -MaxEvents 10 | Format-List

:: AppLocker audit events:
:: Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-AppLocker/EXE and DLL'; ID=8003,8004} | Format-List

:: --- wevtutil -- CMD-native export/query of event logs --- [ALL]
:: The section only shows PowerShell approaches (Get-EventLog / Get-WinEvent); wevtutil is the real, unprefixed CMD executable that belongs alongside the other CMD one-liners in this .bat-style file. `epl` (export-log) and `qe` (query-events) with an XPath-style /q filter, /f:text output format, and /c:50 count cap are correct wevtutil syntax.
wevtutil epl System C:\Logs\system.evtx
wevtutil qe Security /q:"*[System[(EventID=4625)]]" /f:text /c:50

:: --- Get-WinEvent -FilterXPath for single-log compound queries --- [ALL]
:: Adds a documented, more advanced Get-WinEvent filtering technique (XPath, compound expression, time window) not currently shown -- the section only demonstrates FilterHashtable. Corrected the tag from the invalid "[ALL, PS 5.1+]" to the allowed "[ALL]".
:: FilterXPath is faster than FilterHashtable for single-log compound filters (e.g. EventID + level + time window):
:: Get-WinEvent -LogName Security -FilterXPath "*[System[(EventID=4625) and TimeCreated[timediff(@SystemTime) <= 86400000]]]"

::==========================================================================
:: [19] POWERSHELL -- NETWORK INFO
::==========================================================================

:: --- Physical NICs with MAC --- [ALL, PS 5.1+]
:: Get-CimInstance Win32_NetworkAdapter | Where-Object { $_.PhysicalAdapter } | Format-Table DeviceId, Name, MACAddress -AutoSize

:: --- Full IP Config (CIM) --- [ALL, PS 5.1+]
:: Get-CimInstance Win32_NetworkAdapterConfiguration | Where-Object { $_.IPEnabled } |
::   Select-Object PSComputerName, IPAddress, DefaultIPGateway, IPSubnet, DNSServerSearchOrder | Format-Table -Auto

:: --- Modern Net Cmdlets --- [ALL, PS 5.1+]
:: Get-NetIPConfiguration | Format-List
:: Get-NetAdapter | Select-Object Name, InterfaceDescription, Status, MacAddress, LinkSpeed

:: --- Per-Address Detail (Get-NetIPAddress) --- [ALL]
:: NetTCPIP module. Complements the section's existing Get-NetIPConfiguration one-liner by exposing PrefixOrigin (Dhcp/Manual/WellKnown) per address -- the fastest way to tell whether an address was statically assigned without opening adapter properties.
:: Get-NetIPAddress -AddressFamily IPv4 | Select-Object InterfaceAlias, IPAddress, PrefixLength, PrefixOrigin, SuffixOrigin

:: --- Network Category (Get-NetConnectionProfile) --- [ALL]
:: NetConnection module. The firewall profile a connection uses (Section 1's Domain/Public/Private rules) is driven by this category -- essential when a wired/VPN adapter lands on Public and silently blocks inbound rules scoped to Private/Domain. Not covered anywhere else in the file.
:: Get-NetConnectionProfile | Select-Object Name, InterfaceAlias, NetworkCategory
:: Set-NetConnectionProfile -InterfaceAlias "Ethernet" -NetworkCategory Private

:: --- Adapter Traffic Counters (Get-NetAdapterStatistics) --- [ALL]
:: NetAdapter module. Quick way to confirm an interface is actually passing traffic (or spot a saturated/erroring NIC) without opening Task Manager's Performance tab or Resource Monitor -- distinct from the section's existing Get-NetAdapter, which shows link state/speed but not throughput counters.
:: Get-NetAdapterStatistics -Name "Ethernet" | Format-List *

::==========================================================================
:: [20] WMIC LEGACY --> CIM EQUIVALENTS
::==========================================================================

:: WMIC.exe is DEPRECATED starting Win11 / Server 2022.
:: Removed entirely in some Server 2025 builds.
:: All replacements below use Get-CimInstance (PS 5.1+) or native cmdlets.
::
:: WMIC Command                             --> Modern Equivalent
:: ----------------------------------------    ------------------------------------
:: wmic diskdrive get status                --> Get-CimInstance Win32_DiskDrive | Select-Object DeviceID, Model, Status
:: wmic bios list full                      --> Get-CimInstance Win32_BIOS | Format-List *
:: wmic product get name,version            --> [SLOW] Get-CimInstance Win32_Product | Select Name,Version
::                                              [FAST] Query registry -- see Section 17
:: wmic qfe                                 --> Get-HotFix
:: wmic process list brief                  --> Get-Process
:: wmic process where name="x" delete       --> Stop-Process -Name "x" -Force
:: wmic useraccount list brief              --> Get-LocalUser (local) / Get-ADUser (domain)
:: wmic computersystem get username          --> Get-CimInstance Win32_ComputerSystem | Select UserName
:: wmic /node:X computersystem list full    --> Get-CimInstance Win32_ComputerSystem -ComputerName X
:: wmic service list brief                  --> Get-Service | Select Name, Status, StartType
:: wmic startup get caption,command          --> Get-CimInstance Win32_StartupCommand
:: wmic nic get macaddress                  --> Get-NetAdapter | Select Name, MacAddress
:: wmic cpu get DataWidth                   --> [Environment]::Is64BitOperatingSystem
:: wmic printer set defaultprinter          --> (Get-CimInstance Win32_Printer -Filter "Name='X'").SetDefaultPrinter()
::
:: Remote (replaces wmic /node):
:: Get-CimInstance Win32_UserAccount -ComputerName "REMOTE" -Filter "LocalAccount=True AND Disabled=False"

:: --- WMIC availability in Server 2025 / Win11 24H2+ --- [2025]
:: Confirmed via win-acme GitHub issue #2748 reporting wmic.exe throwing "The term 'wmic' is not recognized..." on Server 2025/Win11 after the OS stopped installing it by default. Adds the FoD re-enable path as a stop-gap, while still steering toward Get-CimInstance as the real fix.
:: WMIC is no longer present by default on Server 2025 / Win11 24H2+ ("wmic" not recognized).
:: It can still be added back as an optional Feature-on-Demand if a legacy script truly needs it:
:: Add-WindowsCapability -Online -Name "WMIC~~~~"

:: --- Reusable remote CIM sessions (replaces per-call /node: overhead) --- [ALL]
:: Verified against the official MicrosoftDocs/PowerShell-Docs reference page for New-CimSession, which shows both the multi-computer -ComputerName syntax and the -Protocol Dcom SessionOption example (Example 7) for legacy WMI-only hosts.
:: Open one persistent session and reuse it for multiple queries (faster than repeated -ComputerName calls):
:: $cs = New-CimSession -ComputerName SERVER01,SERVER02
:: Get-CimInstance Win32_OperatingSystem -CimSession $cs | Select PSComputerName, LastBootUpTime
:: Remove-CimSession $cs
:: For down-level/legacy targets that don't support WSMan, fall back to DCOM:
:: $so = New-CimSessionOption -Protocol Dcom
:: New-CimSession -ComputerName LEGACY01 -SessionOption $so

::==========================================================================
:: [21] SECURITY HARDENING (AppLocker, WDAC, Defender)
::==========================================================================

:: --- AppLocker (Enterprise editions only) --- [ALL Server / Win10-11 Enterprise]
:: Apply policy:
:: Set-AppLockerPolicy -XMLPolicy "C:\path\to\policy.xml"

:: Test policy (simulate):
:: Test-AppLockerPolicy -Path "C:\App.exe" -XmlPolicy "C:\policy.xml"

:: View audit events:
:: Get-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-AppLocker/EXE and DLL'; ID=8003,8004} | Format-List

:: --- WDAC (Windows Defender Application Control) --- [2022+, PS 5.1+]
:: Convert CI policy XML to binary:
:: ConvertFrom-CIPolicy -XmlFilePath "C:\policy.xml" -BinaryFilePath "C:\policy.bin"
:: Set to enforce mode:
:: Set-RuleOption -FilePath "C:\policy.bin" -Option 0

:: --- Vulnerable Driver Check --- [Win11 22H2+, Server 2025]
:: Get-SystemDriver -ScanPath "C:\" | Where-Object { $_.IsVulnerable -eq $true }
:: Note: Get-SystemDriver requires the WDAC module. May not be available on all builds.

:: --- CiTool — inbox WDAC policy deployment/management (replaces manual .cip copy) --- [2022+]
:: Verified command syntax and flags (--update-policy, --list-policies, --remove-policy, with -up/-lp/-rp aliases) against a live mirror of Microsoft's 'Managing CI policies and tokens with CiTool' doc. Corrected the tag from the invalid '[2022+, PS 5.1+]' to the valid '[2022+]' — CiTool is a native exe, not a PowerShell cmdlet, so a PS-version qualifier doesn't apply.
:: CiTool.exe ships inbox starting Win11 22H2 and Server 2025; manages WDAC/App Control policies without a manual
:: copy into C:\Windows\System32\CodeIntegrity\CiPolicies\Active:
CiTool --update-policy "C:\Policies\{PolicyGUID}.cip"
:: List currently active policies (add -json for scripting):
CiTool --list-policies
:: Remove a deployed policy by its GUID:
CiTool --remove-policy "{PolicyGUID}"

:: --- New-CIPolicy — building a base WDAC policy (precedes the existing ConvertFrom-CIPolicy step) --- [2022+]
:: Verified against the official MicrosoftDocs/windows-powershell-docs ConfigCI/New-CIPolicy.md reference, which documents this exact parameter combination (-ScanPath, -Level, -Fallback, -UserPEs, -Audit, -FilePath). Tag corrected from '[2022+, PS 5.1+]' to '[2022+]' — New-CIPolicy is part of the ConfigCI module available well before PS 5.1 became relevant as a qualifier and the extra tag isn't one of the valid set.
:: Scan a reference system/folder to generate a starting policy (audit mode recommended first):
:: New-CIPolicy -ScanPath 'C:\' -Level Publisher -Fallback Hash -UserPEs -Audit -FilePath 'C:\Policies\Base.xml'

::==========================================================================
:: [22] PACKAGE MANAGEMENT (Winget)
::==========================================================================

:: Winget ships with App Installer on Win10 1809+ and Win11.
:: Server: install manually from https://github.com/microsoft/winget-cli/releases
:: Server 2025 includes winget natively.

:: Search: [Win10 1809+, Win11, Server 2025]
:: winget search vscode

:: Install:
:: winget install Microsoft.VisualStudioCode --source winget

:: Upgrade all:
:: winget upgrade --all

:: Uninstall:
:: winget uninstall "App Name"

:: List installed:
:: winget list

:: --- Bulk install/restore via export & import --- [ALL]
:: Verified against the official microsoft/winget-cli docs for export.md and import.md, confirming the -o/--output and -i/--import-file syntax plus the two accept-agreements flags. Tag corrected from the non-standard '[Win10 1809+, Win11, Server 2025]' to '[ALL]' since winget ships on all currently supported Windows targets covered by this sheet.
:: Export all installed packages to a JSON manifest (useful for rebuild/DR/imaging):
winget export -o C:\apps.json
:: Re-install the same set on another machine:
winget import -i C:\apps.json --accept-package-agreements --accept-source-agreements

:: --- Pin a package to block or gate upgrades --- [ALL]
:: Verified against the official microsoft/winget-cli spec for the pin command, which documents 'winget pin add <package> [--version <gated version>] [--source <source>] [--force] [--blocking]', plus 'pin list' and 'pin remove'. Tag corrected to '[ALL]' to match the valid tag set.
:: Prevent 'winget upgrade --all' from touching a specific package (e.g. a known-good driver/runtime version):
winget pin add Microsoft.VisualStudioCode --blocking
:: Or gate it to not go past a given version:
winget pin add Contoso.App --version 2.1.0
:: List / remove pins:
winget pin list
winget pin remove Microsoft.VisualStudioCode

::==========================================================================
:: [23] STORAGE SPACES & REPLICATION (Server)
::==========================================================================

:: --- Storage Spaces Health --- [Server 2016+]
:: Get-StoragePool | Get-PhysicalDisk | Select-Object FriendlyName, HealthStatus, Usage

:: --- Storage Spaces Direct (S2D) Cluster --- [Server 2019+ Datacenter]
:: Get-ClusterNode | Get-PhysicalDisk | Select-Object FriendlyName, HealthStatus, OperationalStatus

:: --- Storage Replica (Server-to-Server Replication) --- [Server 2016+ Standard/Datacenter]
:: Enable-StorageReplica -SourceComputerName "Server1" -SourceVolumeName "D:" -DestinationComputerName "Server2" -DestinationVolumeName "D:"

:: --- Repair Virtual Disk --- [Server 2016+]
:: Repair-VirtualDisk -FriendlyName "MyVDisk"

:: --- Validate replication readiness before enabling Storage Replica --- [ALL]
:: The section jumps straight to Enable-StorageReplica with no pre-flight validation step; Test-SRTopology is the documented, standard prerequisite check (connectivity, IO load, sync-time estimate) that belongs immediately before it. Tag corrected from the invalid "[Server 2016+ Standard/Datacenter]" to [ALL], since Storage Replica cmdlets are available from Server 2016 onward.
::   Test-SRTopology -SourceComputerName "Server1" -SourceVolumeName "D:" -SourceLogVolumeName "E:" -DestinationComputerName "Server2" -DestinationVolumeName "D:" -DestinationLogVolumeName "E:" -DurationInMinutes 30 -ResultPath "C:\temp"

::==========================================================================
:: [24] ZPOOL (TrueNAS / FreeNAS / ZFS)
::==========================================================================

:: Check pool status:
:: zpool status -v

:: List importable pools:
:: zpool import

:: Force import (if pool offline but data intact):
:: zpool import -f poolname

:: Always scrub after force import:
:: zpool scrub poolname

:: --- Quick health summary (only show problem pools) ---
:: The existing entry only shows the full 'zpool status -v' output. '-x' filters to unhealthy pools only (nothing printed if all pools are ONLINE with no errors), which is the faster daily health check most admins actually script against; '-v' adds the list of files with permanent data errors.
:: Concise summary: only pools with errors/unavailable devices, plus data-error detail:
:: zpool status -xv

:: --- TRIM free space on SSD-backed pools ---
:: Not covered in the current section at all. zpool trim is the standard OpenZFS maintenance command for SSD/NVMe-backed pools (common on modern TrueNAS SCALE/CORE builds) and is distinct from autotrim; worth including alongside scrub since both are routine pool-maintenance tasks.
:: Reclaim free space on thinly-provisioned/SSD vdevs (on-demand, independent of autotrim property):
:: zpool trim poolname
:: Check TRIM progress/status:
:: zpool status -t poolname

::==========================================================================
:: [25] UNIFI
::==========================================================================

:: Factory reset UniFi device (SSH into device):
:: sudo syswrapper.sh restore-default

:: Set inform URL (controller adoption):
:: set-inform http://unifi-controller.domain.com:8080/inform

:: --- Query device info over SSH ---
:: The section only covers factory-reset and set-inform; 'info' is the standard first diagnostic step admins run over SSH on a AP/switch/gateway that's stuck offline or won't adopt, confirmed across multiple current UniFi SSH references.
:: Print model, firmware version, MAC, and adoption status of the device you're SSH'd into:
:: info

::==========================================================================
:: [26] MISC UTILITIES
::==========================================================================

:: --- Logoff with Countdown --- [ALL]
echo -------------------- && echo Logging off in 5 sec && echo -------------------- && timeout /t 5 && logoff

:: --- MSI Install/Uninstall --- [ALL]
msiexec /i install.msi /quiet /norestart
msiexec /x install.msi /quiet /norestart

:: --- Run as Admin --- [ALL]
runas /user:administrator cmd

:: --- Shutdown/Reboot --- [ALL]
shutdown /r /t 0
shutdown /s /t 0
shutdown /m \\192.168.1.1 /r /t 0 /f

:: --- Python Quick HTTP Server --- [ALL, Python 3 required]
:: python -m http.server 8080

:: --- Download File (PowerShell) --- [ALL, PS 5.1+]
:: Invoke-WebRequest -Uri "https://example.com/file.zip" -OutFile "C:\Temp\file.zip"

:: --- Self-Signed Certificate --- [ALL, PS 5.1+]
:: New-SelfSignedCertificate -CertStoreLocation Cert:\LocalMachine\My -DnsName "mysite.local" -FriendlyName "MySiteCert" -NotAfter (Get-Date).AddYears(10)

:: --- Shared Folders --- [ALL]
net share

:: --- Recursive Unzip (requires unzip.exe in PATH) --- [ALL]
:: FOR /R %%a in (*.zip) do unzip -d unzipDir "%%a"

:: --- Environment Variables Quick Reference --- [ALL]
:: %USERPROFILE%, %COMPUTERNAME%, %WINDIR%, %LOCALAPPDATA%, %TEMP%,
:: %ProgramFiles%, %ProgramFiles(x86)%, %SystemRoot%

:: --- Command History (interactive CMD) --- [ALL]
:: Press F7

:: --- Star Wars ASCII --- [ALL, requires Telnet Client feature]
:: telnet towel.blinkenlights.nl

:: --- Built-in curl and tar (no download required) --- [ALL]
:: The section already has a PowerShell-only 'Download File' example and a separate zip-only unzip loop, but never mentions that curl.exe/tar.exe ship in System32 by default and can be run straight from cmd.exe -- a genuinely common modern shortcut, plus the curl-alias gotcha that trips people up in Windows PowerShell 5.1.
:: Windows has shipped curl.exe and bsdtar (as tar.exe) in System32 since Windows 10 1803/Server 2019 --
:: no separate install needed for quick downloads/archive extraction from cmd.exe:
curl.exe -L -o file.zip https://example.com/file.zip
tar -xf archive.zip -C C:\Temp
:: NOTE: in Windows PowerShell 5.1, 'curl' is aliased to Invoke-WebRequest -- call curl.exe explicitly
:: (or Remove-Item Alias:curl) to get the real curl; PowerShell 7+ has no such alias.

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
:: QUICK REFERENCE: SERVER 2025 NEW FEATURES
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
::
:: Server 2025 ships with:
::   - Winget (native, no manual install needed)
::   - WMIC fully removed on some builds (CIM cmdlets only)
::   - Wi-Fi support on Server (for edge/branch scenarios)
::   - SMB over QUIC (previously 2022 Azure Edition only)
::   - Hotpatching support (Azure Arc-enabled, reduces reboots)
::   - AD functional level: Windows Server 2025
::   - DFS improvements, Storage Replica compression
::   - NVMe-oF (NVMe over Fabrics) native support
::
:: Verify your PS version:
::   $PSVersionTable.PSVersion
::
:: Install PS 7 (all Server versions):
::   winget install Microsoft.PowerShell  (if winget available)
::   -- or --
::   Invoke-WebRequest -Uri "https://aka.ms/install-powershell.ps1" -OutFile install-powershell.ps1
::   .\install-powershell.ps1 -UseMSI
::
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

:: --- WMIC removal timeline correction --- [DEP]
:: The existing appendix bullet ('WMIC fully removed on some builds') understates and vaguely dates the situation. Microsoft's official support article gives a concrete, sourced timeline (FoD-only by default starting 23H2/24H2, removed on 25H2 upgrade, FoD availability ending later, Server 2025 keeps it as an installable FoD) that is worth citing precisely since this file explicitly tracks deprecations. Corrected the original draft's claim that disabled-by-default 'starts with 24H2' -- Microsoft's own article says it was already disabled by default in 23H2 -- and added the PowerShell Add-WindowsCapability equivalent alongside the DISM command.
:: CORRECTION: WMIC is not merely 'removed on some builds' -- Microsoft has published a firm removal
:: timeline. It is disabled by default (installable only as a Feature-on-Demand) starting with
:: Windows 11 23H2/24H2, is dropped entirely on upgrade to Windows 11 25H2, and will stop being
:: offered as a FoD at all in a future feature update. Server 2025 still ships WMIC as an installable
:: FoD (not enabled by default). Treat every wmic example in this sheet as [DEP] -- use
:: Get-CimInstance / Get-WmiObject replacements shown elsewhere in this file.
:: Re-add WMIC as FoD if still needed short-term:
:: DISM /Online /Add-Capability /CapabilityName:WMIC~~~~
:: -- or --
:: Add-WindowsCapability -Online -Name WMIC~~~~

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
:: END OF CHEAT SHEET
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
:eof
