---
tags:
  - cybersecurity
  - windows-cli
  - access-control
  - active-directory
  - icacls
  - capstone
status: complete
lab: 1
topic: Department Folder Setup and Access Control
environment: Windows PowerShell / CMD
---

# Capstone Lab 01 – Department Folder Setup and Access Control

## Scenario

> You are a Junior Systems Technician at **VertexSoft**. The HR team needs a secure directory structure for confidential reports. Your entire workflow must use Windows CLI tools only (CMD and PowerShell).

---

## Theory & Background

### What is Access Control?

Access control is the practice of restricting who can read, write, or execute resources on a system. Windows implements this through **Access Control Lists (ACLs)** — a set of rules attached to every file and folder that define what each user or group can do.

### Key Concepts

**RBAC (Role-Based Access Control)** — Users are assigned to roles (groups), and permissions are granted to those roles rather than to individuals. This simplifies management because changing a group's permissions automatically applies to all members.

**Principle of Least Privilege** — Users should only have the minimum permissions they need to do their job. Granting excessive access increases the blast radius if an account is compromised.

**ACL Inheritance** — When a permission is set on a folder, it can automatically propagate to all files and subfolders created within it. This is controlled via the `(OI)` (Object Inherit) and `(CI)` (Container Inherit) flags in `icacls`.

**Windows Permission Precedence** — When conflicting permissions exist, Windows resolves them in this order:
1. Explicit Deny
2. Explicit Allow
3. Inherited Deny
4. Inherited Allow

> 🔗 Reference: https://www.ntfs.com/ntfs-permissions-precedence.htm

---

## Directory Structure

The required folder hierarchy was created with `mkdir`:

```
C:\
└── Departments\
    └── HR\
        └── Reports\
```

![[Pasted image 20260417210807.png]]
**Screenshot evidence:** The `tree` command confirms the structure was created successfully — `C:\Departments` contains `HR`, which contains `Reports`.

cmdlet:
```powershell
mkdir C:\Departments\HR\Reports
```

---

## Step 1 – Create User Groups

Two local security groups were created to implement RBAC:

| Group      | Purpose                                 |
| ---------- | --------------------------------------- |
| `HR_Read`  | Read-only access to HR Reports          |
| `HR_Admin` | Full administrative access to all of HR |
![[Pasted image 20260417172013.png]]
**Screenshot evidence:** `New-LocalGroup` was run for both groups with descriptions, and the output confirms each group was created successfully.

cmdlet:
```powershell
New-LocalGroup -Name "HR_Read"  -Description "Read-only access HR group"
New-LocalGroup -Name "HR_Admin" -Description "Full-Access HR Admin group"
```

---

## Step 2 – Configure ACLs with icacls

### HR_Admin Permissions (applied at `C:\Departments\HR`)

HR_Admin is granted Full Control at the `\HR` folder level so it can manage all HR resources, including subfolders.

![[Pasted image 20260417172236.png|697]]

**Screenshot evidence:** `icacls C:\Departments\HR /grant "HR_Admin:(OI)(CI)(F)"` ran successfully — output confirms `Successfully processed 1 files; Failed processing 0 files`.

cmdlet: 
```cmd
icacls C:\Departments\HR /grant "HR_Admin:(OI)(CI)(F)"
```

| Flag | Meaning |
|---|---|
| `OI` | Object Inherit — permission applies to files inside this folder |
| `CI` | Container Inherit — permission applies to subfolders |
| `F`  | Full Control |

### HR_Read Permissions (applied at `C:\Departments\HR\Reports`)

HR_Read is scoped specifically to `\Reports` — they cannot access other folders within `\HR`.

![[Pasted image 20260417172524.png]]

**Screenshot evidence:** `icacls C:\Departments\HR\Reports\ /grant "HR_Read:(OI)(CI)(RX)"` ran successfully.

```cmd
icacls C:\Departments\HR\Reports\ /grant "HR_Read:(OI)(CI)(RX)"
```

| Flag | Meaning |
|---|---|
| `OI` | Object Inherit |
| `CI` | Container Inherit |
| `RX` | Read and Execute |

---

## Step 3 – Verify Permissions

### Verify HR ACLs

![[Pasted image 20260417172614.png]]
**Screenshot evidence (HR folder):** `icacls C:\Departments\HR` output shows `DESKTOP-NBVFIGL\HR_Admin:(OI)(CI)(F)` highlighted in red — confirming HR_Admin has Full Control with full inheritance.

![[Pasted image 20260417172645.png]]
**Screenshot evidence (Reports folder):** `icacls C:\Departments\HR\Reports` output shows `DESKTOP-NBVFIGL\HR_Read:(OI)(CI)(RX)` highlighted in red — confirming HR_Read has Read+Execute scoped to Reports.

cmdlet:
```powershell
icacls C:\Departments\HR
icacls C:\Departments\HR\Reports
```

### Verify File-Level Inheritance

![[Pasted image 20260417172740.png]]

![[Pasted image 20260417172951.png]]

**Screenshot evidence:** A test file `test.txt` was created in `C:\Departments\HR`. Running `icacls .\test.txt` (and equivalently `icacls C:\Departments\HR\test.txt`) shows that `HR_Admin:(I)(F)` is already present — the `(I)` flag indicates this was **automatically inherited** from the `\HR` folder's `(OI)` setting, without any explicit file-level grant being needed.

cmdlet
```powershell
New-Item test.txt                           # Create test file in C:\Departments\HR
icacls .\test.txt                           # Verify using relative path
icacls C:\Departments\HR\test.txt           # Or using absolute path
```

---

## Step 4 – Conflicting Permissions Demonstration

This section simulates and resolves a conflict between a user's individual permissions and their group permissions — demonstrating the Windows permission precedence hierarchy in practice.

### #1 – Create testuser and add to HR_Read

![[Pasted image 20260417173027.png]]
**Screenshot evidence:** `net user testuser /add` was run, followed by `net user` to confirm `testuser` appears in the account list (highlighted in red). Then `net localgroup HR_Read testuser /add` added the user to the group.

cmdlet: 
```cmd
net user testuser <password> /add
net localgroup HR_Read testuser /add
```

### #2 – Create testfile.txt and verify inherited permissions

![[Pasted image 20260417173121.png|697]]

![[Pasted image 20260417173243.png]]

**Screenshot evidence:** `New-Item "C:\Departments\HR\Reports\testfile.txt"` created the file. Running `icacls "C:\Departments\HR\Reports\"` confirms `HR_Read:(OI)(CI)(RX)` is set on the folder, and `icacls "C:\Departments\HR\Reports\testfile.txt"` shows `HR_Read:(I)(RX)` — the file correctly inherited Read+Execute from its parent folder.

cmdlet:
```powershell
New-Item "C:\Departments\HR\Reports\testfile.txt"
Add-Content "C:\Departments\HR\Reports\testfile.txt" "Secret HR document"
icacls "C:\Departments\HR\Reports\"
icacls "C:\Departments\HR\Reports\testfile.txt"
```

### #3 – Verify testuser can read the file (before DENY)

![[Pasted image 20260417173342.png]]
**Screenshot evidence:** `runas /user:testuser powershell` was used to launch PowerShell as testuser. After confirming identity with `whoami`, running `Get-Content "C:\Departments\HR\Reports\testfile.txt"` successfully returned `"Secret HR document"` — confirming testuser inherits HR_Read's RX permission.

cmdlet: 
```powershell
runas /user:testuser powershell
whoami                                              # Verify: desktop-nbvfigl\testuser
Get-Content "C:\Departments\HR\Reports\testfile.txt"
```

### #4 – Apply Explicit DENY to testuser

![[Pasted image 20260417173516.png|697]]
**Screenshot evidence:** `icacls "C:\Departments\HR\Reports\" /deny "testuser:(OI)(CI)(R)"` was run. The subsequent `icacls` output clearly shows two conflicting entries: `testuser:(DENY)(R)` above `HR_Read:(OI)(CI)(RX)` — the DENY is now explicit and will take precedence.

cmdlet: 
```cmd
icacls "C:\Departments\HR\Reports\" /deny "testuser:(OI)(CI)(R)"
icacls "C:\Departments\HR\Reports\"
```

### #5 – Verify DENY overrides group Allow

![[Pasted image 20260417173706.png]]

**Screenshot evidence:** Back in testuser's PowerShell session, `Get-Content "C:\Departments\HR\Reports\testfile.txt"` now throws `Get-Content : Access is denied` — a `PermissionDenied` / `UnauthorizedAccessException` error. This confirms that an **Explicit Deny overrides an Explicit Allow**, even when the Allow comes from a group membership.

```powershell
Get-Content "C:\Departments\HR\Reports\testfile.txt"
# Result: Access is denied
```

---

## Tools & Alternatives

| Tool Used | Purpose | Alternative |
|---|---|---|
| `icacls` | Set and view NTFS ACLs | `Set-Acl` / `Get-Acl` (PowerShell — more scriptable) |
| `New-LocalGroup` | Create local security groups | `New-ADGroup` (Active Directory domain environments) |
| `net user` / `net localgroup` | Create users, manage group members | `New-LocalUser`, `Add-LocalGroupMember` (PowerShell cmdlets) |
| `runas` | Run process as another user | `Start-Process -Credential` (PowerShell) |

---

## Cheatsheet – Key Commands

```powershell
# --- Folder Structure ---
mkdir C:\Departments\HR\Reports
tree C:\Departments                                    # View directory tree

# --- Group Management ---
New-LocalGroup -Name "GroupName" -Description "Desc"
net localgroup GroupName username /add                 # Add user to group (CMD)
Add-LocalGroupMember -Group "GroupName" -Member "user" # PowerShell equivalent

# --- User Management ---
net user username password /add                        # Create local user (CMD)
New-LocalUser -Name "username" -Password $pwd          # PowerShell equivalent
net user                                               # List all users

# --- ACL Configuration (icacls) ---
icacls <path> /grant "Group:(OI)(CI)(F)"              # Full control + inheritance
icacls <path> /grant "Group:(OI)(CI)(RX)"             # Read + Execute + inheritance
icacls <path> /deny  "user:(OI)(CI)(R)"               # Explicit deny with inheritance
icacls <path>                                          # View current ACLs

# --- icacls Permission Flags ---
# (OI) = Object Inherit (files inside folder)
# (CI) = Container Inherit (subfolders)
# (I)  = Inherited (read-only indicator in output)
# F    = Full Control
# M    = Modify
# RX   = Read and Execute
# R    = Read
# W    = Write

# --- Verification ---
icacls C:\path\to\folder
icacls C:\path\to\file.txt
icacls .\file.txt                                      # Relative path also works

# --- Run as Another User ---
runas /user:username powershell
whoami                                                 # Confirm current user identity
Get-Content "C:\path\to\file.txt"                      # Test read access
```

---

## Summary

| Requirement | Implemented |
|---|---|
| Create `C:\Departments\HR\Reports` structure | ✅ `mkdir` |
| Create `HR_Read` and `HR_Admin` groups | ✅ `New-LocalGroup` |
| HR_Admin gets Full Control on `\HR` | ✅ `icacls /grant "(OI)(CI)(F)"` |
| HR_Read gets Read+Execute on `\Reports` only | ✅ `icacls /grant "(OI)(CI)(RX)"` |
| Verify with `icacls` | ✅ Demonstrated with screenshots |
| Demonstrate conflicting permissions | ✅ Explicit DENY overrides group ALLOW |
| Apply Least Privilege & RBAC | ✅ Scoped permissions per role |
