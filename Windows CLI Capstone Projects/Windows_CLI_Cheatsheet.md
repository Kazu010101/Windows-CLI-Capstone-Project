# Windows CLI Cheatsheet – VertexSoft Capstone Labs

> Consolidated reference across all 5 capstone labs. Use this as a quick-lookup during CTFs, incident response, or daily sysadmin work.

---

## 📁 File System & Directory Management

```powershell
# Create directories
mkdir C:\Path\To\Folder
New-Item -ItemType Directory -Path "C:\Path" -Force

# Create files
New-Item filename.txt
New-Item C:\Path\script.ps1 -ItemType File

# List directory contents
ls                                          # Alias for Get-ChildItem
Get-ChildItem C:\Path
Get-ChildItem C:\Path -Filter *.log         # Filter by extension
Get-ChildItem C:\Path -Filter *.log -Recurse  # Include subdirectories
Get-ChildItem C:\Path | Sort-Object LastWriteTime

# View directory tree
tree C:\Path

# Read a file
Get-Content C:\Path\file.txt
cat C:\Path\file.txt                        # Alias

# Write / append to a file
Set-Content $file $content                  # Overwrite
Add-Content $file "text to append"          # Append
"text" > file.txt                           # CMD redirect (overwrite)
"text" >> file.txt                          # CMD redirect (append)

# Delete files
Remove-Item "C:\Path\file.txt"
Remove-Item "C:\Path\file.txt" -ErrorAction Stop   # Throw on error
Remove-Item "C:\Path\file.txt" -WhatIf             # Dry run

# Copy / Move
Copy-Item src dst
Move-Item src dst
```

---

## 🔐 ACLs & Permissions (icacls)

```cmd
REM View permissions
icacls C:\Path\Folder
icacls C:\Path\file.txt
icacls .\file.txt                           REM Relative path also works

REM Grant permissions
icacls <path> /grant "Group:(OI)(CI)(F)"    REM Full Control + inheritance
icacls <path> /grant "Group:(OI)(CI)(RX)"   REM Read + Execute + inheritance
icacls <path> /grant "Group:(OI)(CI)(M)"    REM Modify + inheritance
icacls <path> /grant "User:(OI)(CI)(R)"     REM Read + inheritance

REM Deny permissions
icacls <path> /deny "User:(OI)(CI)(R)"      REM Deny Read + inheritance
icacls <path> /deny "Group:(OI)(CI)F"       REM Deny Full Control

REM Remove a permission entry
icacls <path> /remove "Username"

REM Reset permissions to inherited defaults
icacls <path> /reset
```

### icacls Permission Flags

| Flag | Meaning |
|---|---|
| `(OI)` | Object Inherit — applies to files inside the folder |
| `(CI)` | Container Inherit — applies to subfolders |
| `(I)` | Inherited — read-only indicator in output; permission came from parent |
| `(NP)` | No Propagate — don't pass this permission further down |
| `F` | Full Control |
| `M` | Modify (read, write, delete — but not change permissions) |
| `RX` | Read and Execute |
| `R` | Read only |
| `W` | Write only |
| `N` | No access (used in DENY output) |

### Windows Permission Precedence (highest → lowest)

1. Explicit Deny
2. Explicit Allow
3. Inherited Deny
4. Inherited Allow

> ⚠️ An explicit DENY always wins — even against a group ALLOW for the same user.

---

## 👤 User & Group Management

```powershell
# --- Local Users ---
New-LocalUser -Name "username" -Password $pwd -FullName "Full Name" -Description "desc"
Get-LocalUser                                               # List all users
Get-LocalUser -Name "username" -ErrorAction SilentlyContinue  # Check existence
Remove-LocalUser -Name "username"                           # Delete a user
net user                                                    # CMD: list all users
net user username password /add                             # CMD: create user
net user username /delete                                   # CMD: delete user

# --- Local Groups ---
New-LocalGroup -Name "GroupName" -Description "description"
Get-LocalGroup                                              # List all groups
Remove-LocalGroup -Name "GroupName"
net localgroup                                              # CMD: list groups
net localgroup GroupName username /add                      # CMD: add user to group
net localgroup GroupName username /delete                   # CMD: remove user from group

# --- Group Membership ---
Add-LocalGroupMember -Group "GroupName" -Member "username"
Get-LocalGroupMember -Group "GroupName"

# --- Password Handling ---
$pwd = ConvertTo-SecureString "Password123!" -AsPlainText -Force  # From string
$pwd = Read-Host "Enter password" -AsSecureString            # Prompt securely

# --- Run as Another User ---
runas /user:username powershell
whoami                                                      # Confirm current identity
```

---

## ⚙️ Service Management

```powershell
# Query service status
Get-Service <ServiceName>
Get-Service <ServiceName> -RequiredServices      # Show dependencies
sc query <ServiceName>                           # CMD equivalent

# Start / Stop / Restart
Start-Service <ServiceName>
Stop-Service <ServiceName>
Restart-Service <ServiceName>
sc start <ServiceName>                           # CMD
sc stop  <ServiceName>                           # CMD

# Get all service properties / methods
Get-Service <ServiceName> | Get-Member

# Access service status as a value
$status = (Get-Service $service).Status
```

---

## 🔍 Process Management

```powershell
# List all processes
tasklist                                         # CMD
Get-Process                                      # PowerShell

# Filter by name
tasklist | findstr spoolsv
Get-Process | findstr spoolsv

# Kill a process
taskkill /IM processname.exe /F                  # CMD: force kill by image name
taskkill /PID 1234 /F                            # CMD: force kill by PID
Stop-Process -Name processname -Force            # PowerShell
Stop-Process -Id 1234 -Force                     # PowerShell by PID
```

---

## 🌐 Network Diagnostics

```powershell
# IP Configuration
ipconfig                                         # Basic info
ipconfig /all                                    # Full info (DNS, DHCP, MAC)
ipconfig /release                                # Release DHCP lease
ipconfig /renew                                  # Renew DHCP lease
ipconfig /flushdns                               # Clear DNS resolver cache
Get-NetIPConfiguration                           # PowerShell equivalent

# Connectivity Tests
ping 8.8.8.8                                     # Test reachability (ICMP)
ping 8.8.8.8 -n 10                               # Send 10 packets
tracert 8.8.8.8                                  # Trace routing hops (CMD)
Test-Connection -ComputerName 8.8.8.8            # PowerShell ping
Test-NetConnection -ComputerName google.com -Port 443  # Test specific TCP port

# Active Connections
netstat                                          # Active connections
netstat -an                                      # All connections, numeric
netstat -b                                       # Show owning process (admin only)
Get-NetTCPConnection                             # PowerShell equivalent

# DNS Resolution
nslookup google.com                              # Test DNS
nslookup google.com 8.8.8.8                      # Query a specific DNS server
nslookup google.com 1.1.1.1                      # Test with Cloudflare DNS
Resolve-DnsName google.com                       # PowerShell DNS lookup

# Proxy & Firewall
netsh winhttp show proxy                         # Check WinHTTP proxy settings
netsh advfirewall show allprofiles               # View all firewall profiles
Get-NetFirewallProfile                           # PowerShell firewall info

# Set static IP (fix gateway misconfiguration)
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.1.100 `
  -PrefixLength 24 -DefaultGateway 192.168.1.1
```

---

## 📅 Date & Time Arithmetic

```powershell
Get-Date                                         # Current date/time
(Get-Date).AddDays(-14)                          # 14 days ago
(Get-Date).AddDays(90)                           # 90 days in the future
"$(Get-Date)"                                    # Embed timestamp in a string

# File timestamps
(Get-Item $file).LastWriteTime                   # Get last modified date
(Get-Item $file).CreationTime = (Get-Date).AddDays(-20)   # Backdate a file
```

---

## 📜 PowerShell Scripting Patterns

### Input & Output

```powershell
Read-Host "Enter username"                       # Prompt for input (echoed)
Read-Host "Enter password" -AsSecureString       # Prompt with no echo
Write-Host "Message to display"
```

### Conditional Logic

```powershell
if ($condition) { ... }
elseif ($condition) { ... }
else { ... }

# Check if a user exists before creating
if (Get-LocalUser -Name $username -ErrorAction SilentlyContinue)
{
    Write-Host "User already exists"
    exit
}
```

### Loops

```powershell
# For loop (counter-based)
for ($i=1; $i -le 17; $i++) { ... }

# ForEach loop (collection-based)
foreach ($file in $files) { ... }

# Infinite loop (use Ctrl+C to stop)
while ($true)
{
    # do something
    Start-Sleep -Seconds 300
}
```

### Error Handling (try/catch)

```powershell
try
{
    Some-Cmdlet -ErrorAction Stop           # Stop forces catch to trigger on error
    Add-Content $logfile "$(Get-Date) SUCCESS: $item"
}
catch
{
    Write-Host "ERROR DETAILS: $($_.Exception.Message)"
    Add-Content $logfile "$(Get-Date) ERROR: $($_.Exception.Message)"
}
```

### Logging Pattern

```powershell
$logfile = "C:\Logs\script.log"
Add-Content $logfile "$(Get-Date) Action completed for $item"      # Success
Add-Content $logfile "$(Get-Date) ERROR: $($_.Exception.Message)"  # Failure
Get-Content $logfile                                                # Read log
```

### File Discovery Pattern (by age)

```powershell
$folder     = "C:\Logs"
$date_limit = (Get-Date).AddDays(-14)
$files      = Get-ChildItem $folder -Filter *.log

foreach ($file in $files)
{
    if ($file.LastWriteTime -lt $date_limit)
    {
        Remove-Item $file.FullName -ErrorAction Stop
    }
}
```

---

## 🗓️ Task Scheduler

```powershell
# Create and register a scheduled task
$action    = New-ScheduledTaskAction -Execute "Powershell.exe" `
             -Argument "-File C:\Scripts\script.ps1"
$trigger   = New-ScheduleTaskTrigger -Weekly -DaysOfWeek Tuesday -At 10am
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest

Register-ScheduledTask -TaskName "TaskName" `
    -Action $action -Trigger $trigger -Principal $principal `
    -Description "Description of what the task does"

# Manage existing tasks
Get-ScheduledTask                                        # List all tasks
Get-ScheduledTaskInfo -TaskName "TaskName"               # Last run status / next run
Start-ScheduledTask  -TaskName "TaskName"                # Run immediately
Stop-ScheduledTask   -TaskName "TaskName"                # Stop a running task
Unregister-ScheduledTask -TaskName "TaskName" -Confirm:$false  # Delete the task
```

---

## 🚀 Script Execution Policy

```powershell
# Check current policy
Get-ExecutionPolicy
Get-ExecutionPolicy -List                               # Per scope

# Bypass for a single file (lab / admin use only)
powershell -ExecutionPolicy Bypass -File C:\Scripts\script.ps1

# Bypass for current session only
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

# Production setting (requires signed scripts)
Set-ExecutionPolicy AllSigned -Scope LocalMachine
```

---

## 🧪 Useful Exploration Commands

```powershell
# Discover object properties and methods
Get-Service Spooler | Get-Member
Get-Process | Get-Member
Get-Item $file   | Get-Member

# Who am I?
whoami
whoami /groups                                  # Show group memberships

# System info
systeminfo
hostname

# Help
Get-Help <cmdlet> -Examples
Get-Help icacls /?                              # CMD built-in help
```

---

## ⚡ Quick Reference – Reserved PowerShell Variables (Do Not Reuse)

| Reserved | Safe alternative |
|---|---|
| `$home` | `$userHome` |
| `$pwd` | `$currentDir` |
| `$host` | `$hostName` |
| `$error` | `$errResult` |
| `$input` | `$userInput` |
| `$args` | `$scriptArgs` |

---

## 🗂️ Lab Index

| Lab | Topic | Key Tools |
|---|---|---|
| <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_01_Department_Folder_Setup_and_Access_Control.md">Department Folder Setup and Access Control</a> | ACLs & RBAC | `icacls`, `New-LocalGroup`, `net user` |
| <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_02_Process_Crash_and_Service_Restart.md">Process Crash and Service Restart | Service monitoring | `Get-Service`, `Restart-Service`, `tasklist`, `while($true)` |
| <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_03_Network_Access_Denial_Investigation.md">Network Access Denial Investigation | Network triage | `ipconfig`, `ping`, `tracert`, `netstat`, `nslookup` |
| <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_04_Log_Cleanup_Automation.md">Log Cleanup Automation | Scheduled automation | `Get-ChildItem`, `Remove-Item`, `Register-ScheduledTask` |
| <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_05_Onboarding_Automation_Script.md">Onboarding Automation Script | User provisioning | `New-LocalUser`, `Add-LocalGroupMember`, `icacls`, `Read-Host` |
