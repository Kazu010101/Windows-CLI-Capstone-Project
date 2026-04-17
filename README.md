# 🖥️ Windows CLI Capstone Projects

> A series of hands-on labs practising Windows system administration and security tasks using **CMD and PowerShell exclusively** — no GUI allowed.
>
> These labs simulate real-world IT support and junior sysadmin scenarios at a fictional company, **VertexSoft**, and cover access control, process monitoring, network investigation, automation, and user provisioning.

---

## 📋 Lab Overview

| # | Capstone Lab | Key Skills |
|---|-----|-----------|
| 01 | <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_01_Department_Folder_Setup_and_Access_Control.md">Department Folder Setup and Access Control</a> | NTFS ACLs, RBAC, Least Privilege |
| 02 | <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_02_Process_Crash_and_Service_Restart.md">Process Crash and Service Restart | Service management, PS scripting, Monitoring |
| 03 | <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_03_Network_Access_Denial_Investigation.md">Network Access Denial Investigation | Network triage, Incident reporting |
| 04 | <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_04_Log_Cleanup_Automation.md">Log Cleanup Automation | Scripting, Task Scheduler, Log management |
| 05 | <a href="https://github.com/Kazu010101/Windows-CLI-Capstone-Project/blob/main/Windows%20CLI%20Capstone%20Projects/Capstone%20Lab_05_Onboarding_Automation_Script.md">Onboarding Automation Script | User provisioning, Full automation, Error handling |

📎 **[Windows CLI Cheatsheet](./Windows_CLI_Cheatsheet.md)** — consolidated command reference across all labs

---

## 🔐 Lab 01 – Department Folder Setup and Access Control

**Scenario:** The HR team at VertexSoft needs a secure directory structure for confidential reports. Configure folder permissions so that `HR_Admin` has full control and `HR_Read` can only view reports.

**What I did:**
- Created a nested directory structure (`C:\Departments\HR\Reports`) using CLI
- Created two local security groups (`HR_Admin`, `HR_Read`) to implement RBAC
- Configured NTFS ACLs using `icacls` with inheritance flags `(OI)(CI)`
- Verified permissions and simulated a conflicting permission scenario to demonstrate Windows permission precedence (Explicit Deny > Explicit Allow)

**Security concepts applied:** Role-Based Access Control (RBAC), Principle of Least Privilege, NTFS permission inheritance, Explicit vs. Inherited permissions

**Tools:** `icacls`, `New-LocalGroup`, `net user`, `net localgroup`, `runas`

👉 **[View Full Lab Notes](./Lab_01_Department_Folder_Setup_and_Access_Control.md)**

---

## ⚙️ Lab 02 – Process Crash and Service Restart

**Scenario:** A critical service on an app server keeps crashing. Investigate the process, restart it, and write a monitoring script that automatically recovers the service every 5 minutes.

**What I did:**
- Identified the `spoolsv.exe` process using `tasklist` and confirmed via `Get-Service`
- Simulated a crash using `taskkill /F` and verified the PID changed after restart
- Wrote a PowerShell monitoring script (`scriptcheck.ps1`) using an infinite loop (`while ($true)`) with a 5-minute `Start-Sleep` interval
- Script automatically restarts the service if not running and logs each event with a timestamp
- Verified service dependencies using `Get-Service -RequiredServices`

**Security concepts applied:** Service dependency mapping, automated recovery, audit logging, PID-based restart verification

**Tools:** `Get-Service`, `Restart-Service`, `tasklist`, `taskkill`, `Add-Content`, `Start-Sleep`

👉 **[View Full Lab Notes](./Lab_02_Process_Crash_and_Service_Restart.md)**

---

## 🌐 Lab 03 – Network Access Denial Investigation

**Scenario:** Users in the Sales department cannot access the internet. Investigate the root cause, collect evidence, and produce a report for both technical and non-technical stakeholders.

**What I did:**
- Used `ipconfig` to identify a misconfigured default gateway (`192.168.1.200` instead of the actual router)
- Confirmed internet failure with `ping 8.8.8.8` (100% packet loss) and `tracert` (route fails at hop 1)
- Used `netstat` to confirm no active external connections
- Redirected all diagnostic outputs to a log file using `>>` for evidence preservation
- Ruled out proxy (`netsh winhttp show proxy`) and firewall (`netsh advfirewall show allprofiles`) as contributing factors
- Produced a plain-language incident report for non-technical readers

**Security concepts applied:** OSI Layer 3 triage methodology, evidence collection, root cause analysis, firewall and proxy verification, incident reporting

**Tools:** `ipconfig`, `ping`, `tracert`, `netstat`, `nslookup`, `netsh`

👉 **[View Full Lab Notes](./Lab_03_Network_Access_Denial_Investigation.md)**

---

## 🗂️ Lab 04 – Log Cleanup Automation

**Scenario:** The system accumulates old log files weekly. Automate the deletion of `.log` files older than 14 days, log every deletion, and schedule it to run weekly without manual intervention.

**What I did:**
- Created 17 dummy log files with backdated timestamps to simulate an aged log directory
- Wrote a PowerShell script (`cleanup2.ps1`) using `Get-ChildItem`, date arithmetic (`AddDays(-14)`), and `foreach` to identify and delete old files
- Implemented `try/catch` error handling so deletion failures are logged without crashing the script
- Registered a weekly Task Scheduler job running as SYSTEM using `Register-ScheduledTask`

**Security concepts applied:** Log retention policy, automated remediation, error handling, principle of least-privilege scheduling (SYSTEM context), audit logging

**Tools:** `Get-ChildItem`, `Remove-Item`, `Add-Content`, `Register-ScheduledTask`, `New-ScheduledTaskAction`, `New-ScheduleTaskTrigger`

👉 **[View Full Lab Notes](./Lab_04_Log_Cleanup_Automation.md)**

---

## 👤 Lab 05 – Onboarding Automation Script

**Scenario:** Create a fully automated PowerShell onboarding script for new employees. A single script run should provision the user, assign them to a team, create their home directory, set folder permissions, and produce an audit log.

**What I did:**
- Built an interactive script using `Read-Host` for username and group selection, making it reusable for any new hire
- Implemented duplicate-user detection with `Get-LocalUser -ErrorAction SilentlyContinue` to prevent account collisions
- Automated: user creation (`New-LocalUser`), group assignment (`Add-LocalGroupMember`), home directory creation, and `icacls` permission grants in one script run
- Applied team separation logic: user gets Full Control on their folder, their group gets Modify, the other group is explicitly Denied
- Wrapped all provisioning steps in `try/catch` with `$_.Exception.Message` for actionable error reporting and full audit logging

**Security concepts applied:** AAA framework (Authentication, Authorisation, Accounting), user provisioning security, team-based access separation, error-resilient scripting, audit trail

**Tools:** `New-LocalUser`, `Add-LocalGroupMember`, `icacls`, `Read-Host`, `ConvertTo-SecureString`, `Add-Content`, `try/catch`

👉 **[View Full Lab Notes](./Lab_05_Onboarding_Automation_Script.md)**

---

## 🔧 Tools & Technologies Used

![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?style=flat&logo=powershell&logoColor=white)
![Windows](https://img.shields.io/badge/Windows-0078D6?style=flat&logo=windows&logoColor=white)
![CMD](https://img.shields.io/badge/CMD-4D4D4D?style=flat&logo=windowsterminal&logoColor=white)

| Category | Tools |
|---|---|
| Access Control | `icacls`, `New-LocalGroup`, `net localgroup` |
| User Management | `New-LocalUser`, `net user`, `runas` |
| Process & Services | `Get-Service`, `tasklist`, `taskkill`, `Restart-Service` |
| Networking | `ipconfig`, `ping`, `tracert`, `netstat`, `nslookup`, `netsh` |
| Automation | PowerShell scripting, `Register-ScheduledTask`, `try/catch` |
| Logging | `Add-Content`, output redirection (`>`, `>>`) |

---

## 📎 Quick Reference

📄 **[Windows CLI Cheatsheet](./Windows_CLI_Cheatsheet.md)** — all commands from every lab in one place, organised by category

---

*← Back to [Portfolio Home](../README.md)*
