---
tags:
  - cybersecurity
  - windows-cli
  - automation
  - powershell-scripting
  - user-provisioning
  - onboarding
  - icacls
  - capstone
status: complete
lab: 5
topic: Onboarding Automation Script
environment: Windows PowerShell
---

# Capstone Lab 05 – Onboarding Automation Script

## Scenario

> Create a PowerShell script that fully automates the onboarding of a new employee. The script must: create a user account, assign them to a security group, set up a home directory, configure appropriate folder permissions using `icacls`, and log all actions.

---

## Theory & Background

### What is User Provisioning?

**User provisioning** is the process of creating and configuring a new user account with the correct access rights, group memberships, and resources. In enterprise environments, this is one of the most repetitive sysadmin tasks — and one of the highest-risk if done manually, as errors can result in over-privileged accounts or missed access logging.

### The AAA Security Framework

Good provisioning scripts implement three core security principles:

| Principle | Applied in this lab |
|---|---|
| **Authentication** | User account created with password via `New-LocalUser` |
| **Authorisation** | Group membership + `icacls` restrict what they can access |
| **Accounting** | Every action logged with timestamp via `Add-Content` |

### Why Script Provisioning?

Manual onboarding is slow, inconsistent, and unauditable. Scripting it provides:
- **Speed:** Multiple tasks (user + group + folder + permissions + log) run in seconds
- **Consistency:** Every new user gets the exact same setup every time
- **Auditability:** The log file acts as a provisioning record
- **Reusability:** `Read-Host` makes the script work for any new username without editing the script

### Reserved PowerShell Variables to Avoid

PowerShell has built-in automatic variables that must not be reused as custom variable names:

| Reserved | Use instead |
|---|---|
| `$home` | `$userHome` |
| `$pwd` | `$currentDir` |
| `$host` | `$hostName` |

Reusing reserved variable names can cause silent bugs or script errors.

> 🔗 Reference: https://github.com/PowerShell/PSScriptAnalyzer/issues/712

---

## Pre-Lab Setup – Create Security Groups

Before running the onboarding script, two groups were created to represent cybersecurity teams:

![[Pasted image 20260417190309.png]]
**Screenshot evidence:** `New-LocalGroup SOC -Description "SOC Team"` and `New-LocalGroup DFIR -Description "DFIR Team"` both ran successfully, with output confirming each group's name and description.

```powershell
New-LocalGroup SOC  -Description "SOC Team"
New-LocalGroup DFIR -Description "DFIR Team"
```

| Group | Description |
|---|---|
| `SOC` | Security Operations Centre team |
| `DFIR` | Digital Forensics and Incident Response team |

---

## Step 1 – Create the Script File

![[Pasted image 20260417190352.png]]
![[Pasted image 20260417190418.png]]
**Screenshot evidence:** `New-Item C:\Scripts\onboarding.ps1 -ItemType File` creates the script, and `edit C:\Scripts\onboarding.ps1` opens it in the Windows editor for writing.

```powershell
New-Item C:\Scripts\onboarding.ps1 -ItemType File
edit C:\Scripts\onboarding.ps1
```

---

## Step 2 – Write the Onboarding Script

The full script is split across two screenshots below. The complete script is reconstructed here with inline explanations:
![[Pasted image 20260417190548.png]]
![[Pasted image 20260417190608.png]]
```powershell
# specify log file location
$logfile = "C:\CapstoneLogs\onboarding.log"

# ask for username — Read-Host makes the script reusable for any new employee
$username = Read-Host "Enter new username"

# check if user already exists — prevents duplicate accounts
if (Get-LocalUser -Name $username -ErrorAction SilentlyContinue)
{
    Write-Host "ERROR: Username already exists. Rerun the script with different username"
    Add-Content $logfile "$(Get-Date) Onboarding FAILED - username $username already exists"
    exit
}

# ask new user's group (interactive menu)
Write-Host "Select User's Group: "
Write-Host "1 - SOC"
Write-Host "2 - DFIR"

$choice = Read-Host "Enter 1 or 2 only: "

if ($choice -eq "1")
{
    $group = "SOC"
}
elseif ($choice -eq "2")
{
    $group = "DFIR"
}
else
{
    Write-Host "ERROR - Invalid selection"
    Add-Content $logfile "(Get-Date) Onboarding FAILED - invalid group selection"
    exit
}

try
{
    # create password as SecureString (required by New-LocalUser)
    $password = ConvertTo-SecureString "Onboarding123!" -AsPlainText -Force

    # create the user account
    New-LocalUser $username -Password $password -FullName $username `
        -Description "New staff onboarding" -ErrorAction Stop

    # add user to the selected group
    Add-LocalGroupMember -Group $group -Member $username -ErrorAction Stop

    # create home directory — NOTE: avoid C:\Users\ (restricted by default)
    $userHome = "C:\TeamData\$username"
    New-Item -ItemType Directory -Path $userHome -Force -ErrorAction Stop

    # grant user Full Control over their own home folder
    icacls $userHome /grant "${username}:(OI)(CI)F"

    # grant the group Modify access to the shared team folder
    icacls $userHome /grant "${group}:(OI)(CI)M"

    # deny the other group access entirely
    if ($group -eq "SOC")
    {
        icacls $userHome /deny "DFIR:(OI)(CI)F"
    }
    else
    {
        icacls $userHome /deny "SOC:(OI)(CI)F"
    }

    Write-Host "User onboarding is completed successfully"
    Add-Content $logfile "$(Get-Date) ONBOARDING SUCCESS - User $username is created in $group group"
}
catch
{
    Write-Host "User Onboarding failed"
    Write-Host "ERROR DETAILS: $($_.Exception.Message)"
    Add-Content $logfile "$(Get-Date) ERROR -Onboarding failed for $username"
}
```

### Key Scripting Decisions

**Why `Read-Host` instead of a hardcoded name?**
`Read-Host` prompts the operator to type a value at runtime. This means the same script can onboard any new employee without modifying the file — just run it and type the name. The `$username` variable is then reused throughout.

**Why `Get-LocalUser -ErrorAction SilentlyContinue` for the duplicate check?**
`Get-LocalUser` throws a terminating error if the user does not exist. `-ErrorAction SilentlyContinue` suppresses that error and returns `$null` instead, which evaluates to false in the `if` statement — so the script only enters the block if the user *does* exist.

**Why not use `C:\Users\` for home directories?**
`C:\Users\` is protected by Windows and restricted by default — scripts attempting to create directories there will receive access errors. `C:\TeamData\` is used as an unrestricted alternative.

> 🔗 Reference: https://stackoverflow.com/questions/61405222

**Why include `Write-Host "ERROR DETAILS: $($_.Exception.Message)"`?**
In the catch block, `$_` is the current error object. `.Exception.Message` gives the human-readable error string. Without this, script failures show no feedback — making debugging difficult.

> 🔗 Reference: https://www.mail-archive.com/packer-tool@googlegroups.com/msg04164.html

---

## Step 3 – Create the Log File

![[Pasted image 20260417191051.png]]
**Screenshot evidence:** `New-Item C:\CapstoneLogs\onboarding.log -ItemType File` creates the log file.

```powershell
New-Item C:\CapstoneLogs\onboarding.log -ItemType File
```

---

## Step 4 – Execute and Verify

### Run the Script

![[Pasted image 20260417191629.png]]
**Screenshot evidence:** `.\onboarding.ps1` is run. The interactive prompts appear:
1. `Enter new username:` → operator types `Luke`
2. `Select User's Group:` → `1 - SOC` shown, operator selects `1`

The script outputs:
- Three lines of `Successfully processed 1 files; Failed processing 0 files` (one per `icacls` call)
- `User onboarding is completed successfully`

The directory info below confirms `C:\TeamData\Luke` was created with the correct timestamps.

```powershell
.\onboarding.ps1
# Enter new username: Luke
# Enter 1 or 2 only: 1
```

### Verify the Onboarding Log

![[Pasted image 20260417191804.png]]
**Screenshot evidence:** `Get-Content C:\CapstoneLogs\onboarding.log` shows the full log history. Previous runs show failed attempts (other test usernames), but the final entry at the bottom (highlighted in red) reads:
`03/16/2026 01:07:29 ONBOARDING SUCCESS - User Luke is created in SOC group`

```powershell
Get-Content C:\CapstoneLogs\onboarding.log
```

### Verify icacls Permissions

![[Pasted image 20260417191926.png]]
**Screenshot evidence:** `icacls C:\TeamData\Luke\` confirms the correct permission structure (highlighted in red):

| Principal | Permission | Meaning |
|---|---|---|
| `DFIR` | `(OI)(CI)(N)` | DFIR is explicitly **denied** all access |
| `SOC` | `(OI)(CI)(M)` | SOC group has **Modify** access |
| `Luke` | `(OI)(CI)(F)` | Luke has **Full Control** over his own folder |

This enforces team separation: SOC members can access Luke's folder for collaboration, but DFIR members are completely blocked.

```powershell
icacls C:\TeamData\Luke\
```

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative / Production Tool |
|---|---|---|
| `New-LocalUser` | Create a local user account | `New-ADUser` (Active Directory) |
| `Add-LocalGroupMember` | Add user to a local group | `Add-ADGroupMember` (Active Directory) |
| `New-Item -ItemType Directory` | Create home directory | `mkdir` |
| `icacls` | Set folder permissions | `Set-Acl` (PowerShell — more granular) |
| `Read-Host` | Interactive input | Script parameters (`param()`) or CSV input |
| `Add-Content` | Append to log | SIEM ingestion (Splunk, Elastic) for enterprise |
| Single PS1 script | Manual onboarding trigger | **Microsoft Identity Manager**, **Azure AD / Entra ID** automation, **ServiceNow** provisioning workflows |

---

## Cheatsheet – Key Commands

```powershell
# --- User Management ---
New-LocalUser -Name "username" -Password $pwd -FullName "Full Name" -Description "desc"
Get-LocalUser                                               # List all local users
Get-LocalUser -Name "username" -ErrorAction SilentlyContinue  # Check if user exists
Remove-LocalUser -Name "username"                           # Delete user

# --- Password Handling ---
$pwd = ConvertTo-SecureString "Password123!" -AsPlainText -Force  # Create SecureString
$pwd = Read-Host "Enter password" -AsSecureString            # Prompt securely (no echo)

# --- Group Management ---
New-LocalGroup -Name "GroupName" -Description "desc"
Add-LocalGroupMember -Group "GroupName" -Member "username"
Get-LocalGroupMember -Group "GroupName"                     # List members

# --- Directory Setup ---
New-Item -ItemType Directory -Path "C:\TeamData\username" -Force

# --- icacls Permissions ---
icacls $path /grant "${username}:(OI)(CI)F"                 # Full control
icacls $path /grant "${group}:(OI)(CI)M"                    # Modify
icacls $path /deny  "OtherGroup:(OI)(CI)F"                  # Deny access
icacls $path                                                # View current ACLs

# --- Interactive Input ---
$username = Read-Host "Enter new username"                  # Prompt for text
$password = Read-Host "Enter password" -AsSecureString       # Prompt with no echo

# --- Duplicate Check Pattern ---
if (Get-LocalUser -Name $username -ErrorAction SilentlyContinue)
{
    Write-Host "User already exists"
    exit
}

# --- Error Handling ---
try
{
    Some-Cmdlet -ErrorAction Stop
    Add-Content $logfile "$(Get-Date) SUCCESS"
}
catch
{
    Write-Host "ERROR DETAILS: $($_.Exception.Message)"
    Add-Content $logfile "$(Get-Date) ERROR: $($_.Exception.Message)"
}

# --- Logging ---
Add-Content $logfile "$(Get-Date) ONBOARDING SUCCESS - User $username in $group group"
Get-Content $logfile                                        # Read the log
```

---

## Summary

| Requirement | Implemented |
|---|---|
| Create new user account | ✅ `New-LocalUser` with SecureString password |
| Add to appropriate security group | ✅ `Add-LocalGroupMember` based on `Read-Host` selection |
| Create home directory | ✅ `New-Item -ItemType Directory` at `C:\TeamData\$username` |
| Grant permissions with `icacls` | ✅ User: Full, Group: Modify, Other group: Deny |
| Log all actions | ✅ `Add-Content` with timestamp for success and failure |
| Input validation (duplicate check) | ✅ `Get-LocalUser -ErrorAction SilentlyContinue` |
| Reusable for any username | ✅ `Read-Host` with `$username` variable |
| Error handling | ✅ `try/catch` with `$_.Exception.Message` |
