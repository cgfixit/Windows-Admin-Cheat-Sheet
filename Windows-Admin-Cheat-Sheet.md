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


::==========================================================================
:: [25] UNIFI
::==========================================================================

:: Factory reset UniFi device (SSH into device):
:: sudo syswrapper.sh restore-default

:: Set inform URL (controller adoption):
:: set-inform http://unifi-controller.domain.com:8080/inform


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

::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
:: END OF CHEAT SHEET
::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
:eof
