---
tags:
  - cybersecurity
  - windows-cli
  - automation
  - powershell-scripting
  - task-scheduler
  - log-management
  - capstone
status: complete
lab: 4
topic: Log Cleanup Automation
environment: Windows PowerShell
---

# Capstone Lab 04 – Log Cleanup Automation

## Scenario

> The system accumulates old log files every week. Create an automated PowerShell script that deletes `.log` files older than 14 days from `C:\CapstoneLogs`, records each deletion, and schedule it to run weekly via Task Scheduler.

---

## Theory & Background

### Why Log Management Matters

Logs are essential for security monitoring, incident response, and system health — but unchecked log accumulation creates its own problems:

- **Disk exhaustion:** Full disks crash services and make systems unresponsive.
- **Compliance:** Many regulations (PCI-DSS, ISO 27001) require log *retention* for a defined period but also mandate *disposal* of logs beyond that window.
- **Forensics:** Keeping logs too long without structure makes investigation harder, not easier.

### Retention Policy

A **log retention policy** defines:
- How long logs must be kept (e.g., 14, 30, 90 days)
- Where logs are stored
- Who is responsible for cleanup

This lab implements a **14-day rolling window** — any log older than 14 days is deleted.

### PowerShell Error Handling: try/catch

PowerShell's `try { } catch { }` block separates the "happy path" from error recovery:
- Code in `try` executes normally
- If any statement in `try` throws a terminating error, execution jumps to `catch`
- `-ErrorAction Stop` on cmdlets converts non-terminating errors into terminating ones so `catch` can intercept them

This pattern is critical for scripts that must log failures rather than silently skip them.

---

## Step 1 – Create Dummy Log Files (Test Data)

Before writing the cleanup script, realistic test data was created: 17 log files with staggered creation dates, so logs 14–17 are older than 14 days.

<img width="554" height="341" alt="image" src="https://github.com/user-attachments/assets/ef129fed-fd32-4620-99f2-b321fcdaafcd" />

**Screenshot evidence:** A PowerShell `for` loop runs from `$i=1` to `17` (highlighted in red). Each iteration creates `C:\CapstoneLogs\log$i.log` with random content (1KB–50KB). The `CreationTime` and `LastWriteTime` of each file are backset by `$i` days using `.AddDays(-$i)`.

```powershell
cd C:\CapstoneLogs
for ($i=1; $i -le 17; $i++)
{
    $file = "C:\CapstoneLogs\log$i.log"

    # create file with random size (1KB–50KB)
    $size    = Get-Random -Minimum 1KB -Maximum 50KB
    $content = "0X" * $size
    Set-Content $file $content

    # set a different date for each file
    $date = (Get-Date).AddDays(-$i)
    (Get-Item $file).CreationTime   = $date
    (Get-Item $file).LastWriteTime  = $date
}
```

### Verify Dummy Logs Creation

<img width="640" height="563" alt="image" src="https://github.com/user-attachments/assets/ec44216d-714b-4a8e-a1c9-7ee9d3a9e5f7" />

**Screenshot evidence:** `ls` (alias for `Get-ChildItem`) in `C:\CapstoneLogs` shows all 17 log files with their `LastWriteTime`. Logs `log14.log` through `log17.log` are highlighted in red — their dates (25/02/2026 to 28/02/2026) are more than 14 days before the lab date of 14/03/2026, confirming they are the deletion targets.

```powershell
ls C:\CapstoneLogs
Get-ChildItem C:\CapstoneLogs | Sort-Object LastWriteTime    # Sort by date for clarity
```

---

## Step 2 – Create the Cleanup Script

### Create the ps1 File

<img width="661" height="198" alt="image" src="https://github.com/user-attachments/assets/32b84ec4-fa0d-4dc0-bfc1-9417a4cfc976" />

**Screenshot evidence:** `New-Item C:\Scripts\cleanup2.ps1 -ItemType File` creates a 0-byte script file dated 14/03/2026.

```powershell
New-Item C:\Scripts\cleanup2.ps1 -ItemType File
```

### Script Contents

<img width="1090" height="756" alt="image" src="https://github.com/user-attachments/assets/bbba1932-8068-425f-bfd3-e827d72a5101" />

**Screenshot evidence:** The full script was entered via `edit cleanup2.ps1`. The key logic sections are:
- `$date_limit = (Get-Date).AddDays(-14)` — calculates the cutoff date
- `Get-ChildItem $folder -Filter *.log` — fetches only `.log` files
- `if ($file.LastWriteTime -lt $date_limit)` — checks if the file is older than 14 days
- `Remove-Item $file.FullName -ErrorAction Stop` — deletes the file; Stop forces catch on error
- `Add-Content $logfile "$(Get-Date) Deleted file: $($file.Name)"` — logs each successful deletion (highlighted in red)
- `catch` block logs the error message instead of crashing the script (highlighted in red)

```powershell
# specify the source folder
$folder = "C:\CapstoneLogs"

# specify the destination path for the logfile
$logfile = "C:\CapstoneLogs\cleanup2.log"

# calculate date from 14 days ago
$date_limit = (Get-Date).AddDays(-14)

# detect all .log files in C:\CapstoneLogs folder
# Get-ChildItem lists files in C:\Logs folder
# -Filter *.log ensures only .log files are returned
$files = Get-ChildItem $folder -Filter *.log

# loop through each logfile found, one file at a time during iteration
foreach ($file in $files)
{
    # conditional check if the last modified date is more than 14 days
    # .LastWriteTime is a property of a file storing the last modified timestamp
    if ($file.LastWriteTime -lt $date_limit)
    {
        # try block attempts to delete the file
        # if error occurs in this block, the operation jumps to the "catch" block
        try
        {
            # delete the file using its absolute path
            # -ErrorAction Stop forces ps. to stop when it encounters an error
            # so that the script jumps to the catch block to be logged
            Remove-Item $file.FullName -ErrorAction Stop

            # if deletion succeeds, log the deletion in cleanup.log
            Add-Content $logfile "$(Get-Date) Deleted file: $($file.Name)"
        }
        # if Remove-Item fails, catch block will run
        catch
        {
            # record the error in the log file instead of stopping the script
            Add-Content $logfile "$(Get-Date) ERROR: unable to delete $($file.Name) - Permission Denied"
        }
    }
}
```

---

## Step 3 – Create the Log File

<img width="759" height="194" alt="image" src="https://github.com/user-attachments/assets/e8616f81-9e8d-4115-b541-d66943b08e7e" />

**Screenshot evidence:** `New-Item C:\CapstoneLogs\cleanup2.log -ItemType File` creates the log file that the script will write to.

```powershell
New-Item C:\CapstoneLogs\cleanup2.log -ItemType File
```

> 💡 **Note:** Pre-creating the log file is optional — `Add-Content` creates the file automatically if it doesn't exist. However, pre-creating it makes it easy to verify the path is correct before running the script.

---

## Step 4 – Execute the Script

### Bypass Execution Policy

<img width="815" height="26" alt="image" src="https://github.com/user-attachments/assets/6c33df07-00c1-42a3-a84e-dd8bbae80ea6" />

**Screenshot evidence:** `powershell -ExecutionPolicy Bypass -File C:\Scripts\cleanup2.ps1` is run to bypass the default policy that blocks unsigned scripts.

```powershell
powershell -ExecutionPolicy Bypass -File C:\Scripts\cleanup2.ps1
```

> ⚠️ **ExecutionPolicy Bypass** should only be used in controlled lab/admin contexts. In production, scripts should be signed with a code-signing certificate and used with `AllSigned` policy instead.

### Alternatively, run directly from PowerShell:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\cleanup2.ps1
```

### Verify the Log

<img width="590" height="149" alt="image" src="https://github.com/user-attachments/assets/27c28241-4c13-4fd3-8a59-441dec8002ce" />

**Screenshot evidence:** After execution, `Get-Content C:\CapstoneLogs\cleanup2.log` shows four timestamped entries (highlighted in red), one for each deleted file:
- `03/14/2026 13:54:48 Deleted file: log14.log`
- `03/14/2026 13:54:48 Deleted file: log15.log`
- `03/14/2026 13:54:48 Deleted file: log16.log`
- `03/14/2026 13:54:48 Deleted file: log17.log`

Logs 1–13 remain because they are within the 14-day retention window.

```powershell
.\cleanup2.ps1
Get-Content C:\CapstoneLogs\cleanup2.log
ls C:\CapstoneLogs                              # Confirm log14-17 are gone
```

---

## Step 5 – Schedule with Task Scheduler

Three PowerShell commands register the task. Each builds on the previous:
<img width="1090" height="36" alt="image" src="https://github.com/user-attachments/assets/0762ca96-e39f-4958-87fa-2487d8cd4a58" />

<img width="1090" height="139" alt="image" src="https://github.com/user-attachments/assets/daa6ce3c-fa35-4834-a626-10bf05069a94" />

**Screenshot evidence:** `$action` and `$trigger` are defined.

**Screenshot evidence:** `$principal` is set to run as SYSTEM (highest privilege), then `Register-ScheduledTask` creates the task. The output confirms: `TaskName: WeeklyLogCleanupTask`, `State: Ready`.

```powershell
# Define what to run
$action = New-ScheduledTaskAction -Execute "Powershell.exe" `
          -Argument "-File C:\Scripts\cleanup2.ps1"

# Define when to run (every Tuesday at 10am)
$trigger = New-ScheduleTaskTrigger -Weekly -DaysOfWeek Tuesday -At 10am

# Define the run-as account (SYSTEM = no user login required)
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest

# Register the task
Register-ScheduledTask -TaskName "WeeklyLogCleanupTask" `
    -Action $action -Trigger $trigger -Principal $principal `
    -Description "Runs a weekly log cleanup script"
```

> 💡 **Why SYSTEM?** Running as SYSTEM means the task executes even when no user is logged in, which is essential for server environments. `RunLevel Highest` ensures it has administrator-level access to delete files.

---

## Q&A – Additional Questions

### What happens if Task Scheduler fails to run?

1. The cleanup script will not execute on schedule.
2. Log files older than 14 days will accumulate beyond the retention policy.
3. Over time, disk space will be consumed and may cause service disruption.

**Mitigation:** Configure Task Scheduler's built-in failure handling:
```powershell
# View task history to detect missed runs
Get-ScheduledTaskInfo -TaskName "WeeklyLogCleanupTask"
```
Or set email notification via Task Scheduler GUI (Action > Send an e-mail) — though this feature is deprecated in newer Windows; use a monitoring tool instead.

### How do you log success or failure of each deletion?

The `try/catch` block already handles this:
- **Success:** `Add-Content $logfile "$(Get-Date) Deleted file: $($file.Name)"`
- **Failure:** `Add-Content $logfile "$(Get-Date) ERROR: unable to delete $($file.Name) - Permission Denied"`
<img width="1090" height="756" alt="image" src="https://github.com/user-attachments/assets/2aba0e24-e38b-4c36-b93f-bb167a5cac56" />


---

## Tools & Alternatives

| Tool Used | Purpose | Alternative |
|---|---|---|
| `Get-ChildItem -Filter *.log` | Find files by extension | `Where-Object { $_.Extension -eq ".log" }` |
| `$file.LastWriteTime` | Get file modification date | `(Get-Item $file).LastWriteTime` |
| `Remove-Item -ErrorAction Stop` | Delete file with error capture | `del` (CMD — no try/catch support) |
| `Add-Content` | Append to log file | `Out-File -Append` |
| `Register-ScheduledTask` | Schedule the script | Task Scheduler GUI (`taskschd.msc`) |
| PowerShell script | Custom log cleanup | **Windows Storage Sense**, **TreeSize Free**, **Log Parser** |

---

## Cheatsheet – Key Commands

```powershell
# --- File Discovery ---
Get-ChildItem C:\Logs -Filter *.log                         # List only .log files
Get-ChildItem C:\Logs -Filter *.log -Recurse                # Include subdirectories
Get-ChildItem C:\Logs | Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-14) }

# --- Date Arithmetic ---
(Get-Date).AddDays(-14)                                     # Date 14 days ago
(Get-Item $file).LastWriteTime                              # Get file's last modified date
(Get-Item $file).CreationTime = (Get-Date).AddDays(-20)    # Backdate a file (for testing)

# --- File Deletion ---
Remove-Item "C:\path\to\file.log"                           # Delete a file
Remove-Item "C:\path\to\file.log" -ErrorAction Stop         # Delete, throw on error
Remove-Item "C:\path\to\file.log" -WhatIf                   # Dry run — shows what would be deleted

# --- Logging ---
Add-Content $logfile "$(Get-Date) Deleted file: $($file.Name)"   # Append with timestamp
Add-Content $logfile "$(Get-Date) ERROR: $($_.Exception.Message)" # Log real error message

# --- Script Execution ---
powershell -ExecutionPolicy Bypass -File C:\Scripts\script.ps1
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass

# --- Task Scheduler ---
$action    = New-ScheduledTaskAction -Execute "Powershell.exe" -Argument "-File C:\path\script.ps1"
$trigger   = New-ScheduleTaskTrigger -Weekly -DaysOfWeek Tuesday -At 10am
$principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest
Register-ScheduledTask -TaskName "TaskName" -Action $action -Trigger $trigger -Principal $principal

# Manage existing tasks
Get-ScheduledTask                                           # List all tasks
Get-ScheduledTaskInfo -TaskName "WeeklyLogCleanupTask"      # Check last run status
Start-ScheduledTask -TaskName "WeeklyLogCleanupTask"        # Run immediately
Unregister-ScheduledTask -TaskName "WeeklyLogCleanupTask"   # Delete the task
```

---

## Summary

| Requirement | Implemented |
|---|---|
| Identify `.log` files older than 14 days | ✅ `$file.LastWriteTime -lt $date_limit` |
| Create dummy logs for testing | ✅ For loop with backdated timestamps |
| Write PowerShell deletion script | ✅ `cleanup2.ps1` with foreach + try/catch |
| Log every deletion and error | ✅ `Add-Content` with timestamp |
| Execute script with Bypass policy | ✅ `powershell -ExecutionPolicy Bypass` |
| Schedule weekly via Task Scheduler | ✅ `Register-ScheduledTask` as SYSTEM |
