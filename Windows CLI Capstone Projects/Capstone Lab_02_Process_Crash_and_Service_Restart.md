---
tags:
  - cybersecurity
  - windows-cli
  - process-management
  - service-monitoring
  - powershell-scripting
  - capstone
status: complete
lab: 2
topic: Process Crash and Service Restart
environment: Windows PowerShell / CMD
---

# Capstone Lab 02 – Process Crash and Service Restart

## Scenario

> A service running on an app server keeps crashing. Use the CLI to investigate the process, restart it, and write a monitoring script that auto-recovers the service every 5 minutes.
>
> **Service used:** `Spooler` (Windows Print Spooler — `spoolsv.exe`) as a safe, stoppable stand-in for a crashing application service.

---

## Theory & Background

### What is a Windows Service?

A Windows Service is a long-running background process managed by the **Service Control Manager (SCM)**. Unlike regular applications, services can start automatically at boot, run without a logged-in user, and are monitored by the OS. In enterprise environments, critical services (web servers, agents, database engines) are commonly run this way.

### Why Services Crash

Common causes of service crashes include:
- Memory leaks or resource exhaustion
- Dependency failure (a required service stops first)
- Corrupt configuration or missing files
- Application bugs or unhandled exceptions

### Process ID (PID) as an Indicator of Restart

When a service is restarted, the OS spawns a **new process** with a **different PID**. Monitoring PID changes is a reliable way to confirm that a restart actually occurred, rather than the service just appearing to be running.

### Service Dependencies

Many services cannot start unless other services are already running. For example, Spooler requires `RPCSS` (Remote Procedure Call) and `http` (HTTP Service). Failing to account for dependencies is a common root cause of restart failures.

---

## Step 1 – Identify the Process

### Verify Spooler is Running

<img width="464" height="134" alt="image" src="https://github.com/user-attachments/assets/b12a27a9-efb4-4388-92f8-fea3b6c0a4b5" />

<img width="431" height="104" alt="image" src="https://github.com/user-attachments/assets/9dd0f23f-8269-45b2-a343-23192f07ed50" />

**Screenshot evidence:** `Start-Service Spooler` followed by `Get-Service Spooler` shows `Status: Running` and `DisplayName: Print Spooler`.

cmdlet:
```powershell
Start-Service Spooler
Get-Service Spooler
```

### Locate the Process in tasklist

<img width="781" height="665" alt="image" src="https://github.com/user-attachments/assets/d2fbbffe-0cda-46a9-9452-e95bfafd482b" />

**Screenshot evidence:** `tasklist` output lists all running processes. `spoolsv.exe` is visible with PID `1492` (highlighted in red), confirming the process is active in memory.

cmdlet:
```cmd
tasklist
tasklist | findstr spoolsv        # Filter for just the Spooler process
```

---

## Step 2 – Terminate the Service

<img width="708" height="45" alt="image" src="https://github.com/user-attachments/assets/76c57fb6-2045-4e7a-aec6-59555a7d5e52" />

**Screenshot evidence:** `taskkill /IM spoolsv.exe /F` was run. The output confirms: `SUCCESS: The process "spoolsv.exe" with PID 1492 has been terminated.`

cmdlet:
```cmd
taskkill /IM spoolsv.exe /F
```

| Flag | Meaning |
|---|---|
| `/IM` | Image name (process name) |
| `/F`  | Force termination — kills the process without waiting |

> ⚠️ **Note:** `Stop-Service Spooler` is the clean, graceful approach. `taskkill /F` simulates a crash by terminating the process abruptly, bypassing service shutdown logic. Use `Stop-Service` in production unless simulating a crash.

---

## Step 3 – Restart and Confirm

<img width="790" height="111" alt="image" src="https://github.com/user-attachments/assets/9b356037-36d8-4b5e-8ec0-661321ac9992" />

**Screenshot evidence:** `Restart-Service Spooler` was run. The follow-up `tasklist | findstr spoolsv` shows `spoolsv.exe` with a **new PID of 7684** (highlighted in red), and `Get-Process | findstr spoolsv` confirms PID 7684 is active. This PID change from 1492 confirms the service was genuinely restarted — not simply still running.

cmdlet: 
```powershell
Restart-Service Spooler
tasklist | findstr spoolsv
Get-Process | findstr spoolsv
Get-Service Spooler                # Confirm Status: Running
```

---

## Step 4 – Service Monitoring Script

### Create the Script File

<img width="691" height="203" alt="image" src="https://github.com/user-attachments/assets/68b2334f-33a0-4eae-9eee-e72059846314" />


**Screenshot evidence:** `New-Item scriptcheck.ps1` was run from `C:\ServiceCrash\`, creating a 0-byte file dated 14/03/2026.

cmdlet: 
```powershell
New-Item C:\ServiceCrash\scriptcheck.ps1
```

### Script Contents

<img width="963" height="491" alt="image" src="https://github.com/user-attachments/assets/10c3944b-6508-4472-84d6-d70672cd4db0" />

**Screenshot evidence:** The script was entered via `edit scriptcheck.ps1`. The key logic is an infinite `while ($true)` loop that checks service status every 300 seconds (5 minutes). If the service is not `Running`, it restarts it and appends a timestamped log entry using `Add-Content` (highlighted in red).

```powershell
# verify the variables of service and logfile location
$service = "Spooler"
$logfile = "C:\ServiceCrash\service_monitor.log"

# trigger an infinite loop to check the status of the service condition
while ($true)
{
    # Get the .Status property from Spooler service ( Get-Service Spooler | Get-Member )
    $status = (Get-Service $service).Status

    # if the status is not equal to Running
    if ($status -ne "Running")
    {
        # Restart Spooler & Append to logfile the date and restarted service name
        Restart-Service $service
        Add-Content $logfile "$(Get-Date) Restarted Service: $service"
    }
    # wait for 5 minutes
    Start-Sleep -Seconds 300
}
```

**Script design notes:**
- `(Get-Service $service).Status` accesses the `.Status` property of the service object — use `Get-Service Spooler | Get-Member` to explore all available properties.
- `Add-Content` **appends** to the log file rather than overwriting it, preserving the full history.
- `$(Get-Date)` embeds a timestamp in the log entry.
- `Start-Sleep -Seconds 300` pauses the loop for 5 minutes between checks, preventing CPU thrashing.

### Verify the Script Works

<img width="695" height="429" alt="image" src="https://github.com/user-attachments/assets/0dea33e4-7212-4525-8da5-69da5dabd1e1" />


**Screenshot evidence:**
1. `Get-Service Spooler` confirms Status is `Stopped`.
2. `.\scriptcheck.ps1` is run. After waiting, `Get-Service Spooler` shows `Running`.
3. `Get-Content .\service_monitor.log` shows the entry: `03/14/2026 02:16:44 Restarted Service: Spooler`.
4. `Get-Process | findstr spoolsv` confirms a new PID of `452` — the service was restarted by the script.

<img width="579" height="151" alt="image" src="https://github.com/user-attachments/assets/643496e8-5614-4e0d-9803-3efcc107d44c" />

5. `edit .\service_monitor.log` confirms the log file contains exactly one timestamped restart entry.

cmdlet: 
```powershell
Stop-Service Spooler                          # Simulate a crash
.\scriptcheck.ps1                             # Run the monitoring script
# Press Ctrl+C after 5 minutes to stop the loop
Get-Service Spooler                           # Should show: Running
Get-Content C:\ServiceCrash\service_monitor.log
Get-Process | findstr spoolsv                 # New PID confirms actual restart
```

---

## Step 5 – Check Service Dependencies

<img width="595" height="128" alt="image" src="https://github.com/user-attachments/assets/67d86013-a6a0-49b3-aa1b-e09d80b0f2d7" />

**Screenshot evidence:** `Get-Service Spooler -RequiredServices` lists two dependencies: `RPCSS` (Remote Procedure Call — Status: Running) and `http` (HTTP Service — Status: Running). Both must be active for Spooler to start.

cmdlet: 
```powershell
Get-Service Spooler -RequiredServices
```

> 💡 **Why this matters:** If the monitoring script attempts `Restart-Service` but a dependency is down, the restart will silently fail or throw an error. A production-grade script should also verify dependencies before attempting restart.

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative / Production Tool |
|---|---|---|
| `Get-Service` | Query Windows service status | `sc query` (CMD), `services.msc` (GUI) |
| `Restart-Service` | Restart a service | `sc stop` + `sc start` (CMD) |
| `Stop-Service` | Stop a service gracefully | `sc stop` |
| `tasklist` | List all running processes | `Get-Process` (PowerShell) |
| `taskkill /F` | Force-terminate a process | `Stop-Process -Force` (PowerShell) |
| Custom PS1 loop | Basic monitoring script | **NSSM** (Non-Sucking Service Manager), **Windows Service Recovery settings**, **Nagios / Zabbix / Datadog** |

---

## Cheatsheet – Key Commands

```powershell
# --- Service Management ---
Get-Service <ServiceName>                           # Check service status
Get-Service <ServiceName> -RequiredServices         # Check dependencies
Start-Service <ServiceName>                         # Start a service
Stop-Service <ServiceName>                          # Stop a service gracefully
Restart-Service <ServiceName>                       # Restart a service

# --- Process Investigation ---
tasklist                                            # List all processes (CMD)
tasklist | findstr <processname>                    # Filter by process name
Get-Process                                         # List all processes (PowerShell)
Get-Process | findstr <processname>                 # Filter by name
taskkill /IM <processname.exe> /F                   # Force-kill by image name (CMD)
Stop-Process -Name <processname> -Force             # Force-kill (PowerShell)

# --- Script / File Creation ---
New-Item <path\script.ps1>                          # Create a new ps1 file
edit <script.ps1>                                   # Open file in notepad-style editor

# --- Script Execution ---
.\scriptname.ps1                                    # Run a local PowerShell script
powershell -ExecutionPolicy Bypass -File <path>    # Bypass execution policy to run

# --- Logging ---
Add-Content $logfile "$(Get-Date) message"          # Append timestamped entry to log
Get-Content <logfile>                               # Read a log file

# --- Useful Exploration ---
Get-Service <Name> | Get-Member                     # See all properties/methods of service object
```

---

## Summary

| Requirement | Implemented |
|---|---|
| Identify process with `tasklist` | ✅ `spoolsv.exe` found at PID 1492 |
| Use `Get-Service` to locate service | ✅ Status confirmed Running |
| Terminate and restart service | ✅ PID change (1492 → 7684) confirmed restart |
| Write a 5-minute monitoring script | ✅ `scriptcheck.ps1` with `while($true)` + `Start-Sleep 300` |
| Log every restart attempt | ✅ `Add-Content` with `$(Get-Date)` timestamp |
| Check service dependencies | ✅ `Get-Service -RequiredServices` shows RPCSS + http |
