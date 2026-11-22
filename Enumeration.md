# Windows Privilege Escalation — Initial Enumeration

Initial enumeration is one of the most important stages of Windows privilege escalation.

After gaining access to a Windows host, the first objective is to understand the environment, identify the current security context, inspect running processes and services, and discover potential misconfigurations.

Proper enumeration helps identify possible privilege escalation vectors before attempting any exploitation.

---

## Contents

1. [System Information](#1-system-information)
2. [User and Group Enumeration](#2-user-and-group-enumeration)
3. [Running Processes and Services](#3-running-processes-and-services)
4. [Installed Software and Patches](#4-installed-software-and-patches)
5. [Network Enumeration](#5-network-enumeration)
   - [Network Connections](#network-connections)
   - [ARP Table](#arp-table-enumeration)
   - [Routing Table](#routing-table-enumeration)
6. [Windows Defender Status](#6-windows-defender-status)
7. [AppLocker Policy Enumeration](#7-applocker-policy-enumeration)

---

## 1. System Information

The first step is to identify the operating system and understand the basic configuration of the target machine.

This information helps determine the available system features, PowerShell version, architecture, and potential vulnerabilities associated with the installed Windows version.

### Commands

**CMD**

```cmd
systeminfo
```

Displays detailed information about the operating system, hardware, system architecture, installed updates, and other configuration details.

```cmd
hostname
```

Displays the computer's hostname.

```cmd
ver
```

Displays the Windows version information.

```cmd
echo %USERNAME%
```

Displays the username of the current session.

**PowerShell**

```powershell
$PSVersionTable
```

Displays the installed PowerShell version and environment information.

```powershell
Get-ComputerInfo
```

Retrieves detailed information about the operating system and computer configuration.

### What to Look For

- **OS Name:** Identify whether the target is a Windows workstation or server.
- **OS Version:** Determine the Windows version and build number.
- **Architecture:** Identify whether the system is x86 or x64.
- **PowerShell Version:** Older versions may lack certain security features and commands.
- **System Configuration:** Identify virtualization and hardware information.
- **Installed Updates:** Check whether security patches have been applied.

> **Warning:** Public Windows exploits may target specific operating system builds or unpatched vulnerabilities. Kernel exploits can cause system instability or a complete system crash. Never run them against production systems without authorization and a clear understanding of their impact.

---

## 2. User and Group Enumeration

Understanding the current user context is essential before investigating privilege escalation opportunities.

The objective is to determine which account is being used, what privileges it possesses, and whether other users or privileged groups exist on the system.

### 2.1. Current User and Privileges

**CMD**

```cmd
whoami
```

Displays the current security context.

```cmd
whoami /priv
```

Displays the privileges assigned to the current user.

```cmd
whoami /groups
```

Displays the security groups to which the current account belongs.

```cmd
whoami /all
```

Displays the current user's security information, including SID, group memberships, and privileges.

### Important Windows Privileges

Pay particular attention to privileges that may become relevant during privilege escalation.

| Privilege                       | Description                                                                                |
| ------------------------------- | ------------------------------------------------------------------------------------------ |
| `SeImpersonatePrivilege`        | Allows a process to impersonate a client after authentication under applicable conditions. |
| `SeAssignPrimaryTokenPrivilege` | Allows a process to assign a primary token to a process.                                   |
| `SeBackupPrivilege`             | Allows bypassing certain file access checks for backup operations.                         |
| `SeRestorePrivilege`            | Allows bypassing certain file access checks for restoration operations.                    |
| `SeDebugPrivilege`              | Allows debugging and accessing processes beyond normal permissions.                        |
| `SeTakeOwnershipPrivilege`      | Allows taking ownership of securable objects.                                              |

**Important:** Having a privilege does not automatically mean that it can be exploited. Exploitability depends on the Windows version, token configuration, service context, and available mitigations.

For example, `SeImpersonatePrivilege` may be relevant to Potato-family privilege escalation techniques, but these techniques are not universally applicable.

### 2.2. Logged-In Users and Local Accounts

**CMD**

```cmd
query user
```

Displays currently logged-in user sessions, including session states and idle times.

```cmd
net user
```

Lists local user accounts.

```cmd
net localgroup
```

Lists local groups configured on the machine.

### 2.3. Group Membership and Details

**CMD**

```cmd
net localgroup administrators
```

Displays members of the local Administrators group.

```cmd
net localgroup "Remote Desktop Users"
```

Displays members of the Remote Desktop Users group.

```cmd
net accounts
```

Displays local account and password policy information, including password expiration and lockout settings.

### What to Look For

- Accounts belonging to privileged local groups.
- Unexpected or non-standard user accounts.
- Users with administrative privileges.
- Accounts with access to Remote Desktop.
- Unusual group memberships.
- Password policy weaknesses.
- Interesting group descriptions or account information.

> **Note:** Other users' Desktop, Documents, and Downloads directories may contain useful files, such as scripts, configuration files, SSH keys, or accidentally exposed credentials. Only inspect files you are authorized to access.

---

# Windows Privilege Escalation

A practical guide to Windows privilege escalation, system enumeration, process communication, service misconfigurations, and common security weaknesses.

# Contents

1. [Initial Enumeration](#1-initial-enumeration)
2. [Communication with Processes](#2-communication-with-processes)
3. [Windows Services](#3-windows-services)
4. [Scheduled Tasks](#4-scheduled-tasks)
5. [File and Directory Permissions](#5-file-and-directory-permissions)
6. [Registry Enumeration](#6-registry-enumeration)
7. [Credential Hunting](#7-credential-hunting)
8. [Token Impersonation](#8-token-impersonation)
9. [Automated Enumeration Tools](#9-automated-enumeration-tools)
10. [Privilege Escalation Workflow](#10-privilege-escalation-workflow)

---

# 3. Windows Services

Windows Services are background processes that perform specific system or application-related tasks.

Unlike regular applications, services can operate without an interactive user session and may run under highly privileged accounts.

Some services operate as:

- `NT AUTHORITY\SYSTEM`
- `NT AUTHORITY\LOCAL SERVICE`
- `NT AUTHORITY\NETWORK SERVICE`
- Dedicated service accounts
- Local or domain administrator accounts

A misconfigured service may create a privilege escalation opportunity if a lower-privileged user can modify its executable, configuration, or service permissions.

## 3.1. Service Enumeration

**CMD**

```cmd
sc query
```

Lists services and their current states.

```cmd
sc query state= all
```

Displays all services, including stopped services.

```cmd
sc qc <SERVICE_NAME>
```

Displays the configuration of a specific service.

**PowerShell**

```powershell
Get-Service
```

Lists installed services.

```powershell
Get-Service | Where-Object {$_.Status -eq "Running"}
```

Displays currently running services.

```powershell
Get-CimInstance Win32_Service |
Select-Object Name, DisplayName, State, StartName, PathName
```

Displays service names, execution accounts, states, and executable paths.

## 3.2. Identifying Privileged Services

One important task is identifying services running under highly privileged accounts.

```powershell
Get-CimInstance Win32_Service |
Where-Object {$_.StartName -eq "LocalSystem"} |
Select-Object Name, State, StartName, PathName
```

This identifies services running as LocalSystem.

**What to Look For**

- Services running as SYSTEM.
- Services running under administrator accounts.
- Unusual third-party services.
- Services with writable executable files.
- Services with weak permissions.
- Unquoted service paths.

**Important:** A service running as SYSTEM is not automatically vulnerable. The important question is whether the current user can influence its execution or configuration.

## 3.3. Unquoted Service Paths

Windows services can contain executable paths with spaces.

Consider the following example:

```text
C:\Program Files\My Application\service.exe
```

If the executable path is not enclosed in quotation marks, Windows may interpret the path ambiguously when resolving the executable.

For example, it may attempt intermediate executable locations such as:

```text
C:\Program.exe
C:\Program Files\My.exe
C:\Program Files\My Application\service.exe
```

If a lower-privileged user can write to an appropriate intermediate directory, this configuration may create a privilege escalation opportunity.

**Enumeration**

```powershell
Get-CimInstance Win32_Service |
Select-Object Name, PathName, StartName
```

**Check directory permissions**

```cmd
icacls "C:\Program Files"
```

**What to Look For**

- Executable paths containing spaces.
- Missing quotation marks.
- Writable intermediate directories.
- Services running with elevated privileges.

**Important:** An unquoted service path alone is not sufficient. The relevant directory must be writable, and the service must execute under a sufficiently privileged account.

## 3.4. Weak Service Permissions

Windows controls service permissions through security descriptors.

If an ordinary user has excessive permissions over a privileged service, they may be able to modify its configuration or behavior.

**Inspect Service Configuration**

```cmd
sc qc <SERVICE_NAME>
```

**Inspect Service Permissions with AccessChk**

```cmd
accesschk.exe -uwcqv "Authenticated Users" *
```

This checks service permissions for the specified security principal.

**What to Look For**

- Permission to change service configuration.
- Permission to start or stop sensitive services.
- Permission to modify service security settings.
- Unexpected write or modification rights.

**Key Takeaway:** Service permissions and executable file permissions are separate security controls. Both should be investigated.

---

# 4. Scheduled Tasks

Windows Task Scheduler allows administrators and applications to execute programs or scripts automatically.

Tasks can run at specific times, during system startup, when a user logs in, or when particular events occur.

Some scheduled tasks operate with elevated privileges, including SYSTEM.

If a lower-privileged user can modify a script or executable used by an elevated scheduled task, it may create a privilege escalation opportunity.

## 4.1. Enumerating Scheduled Tasks

**CMD**

```cmd
schtasks /query
```

Lists scheduled tasks.

```cmd
schtasks /query /fo LIST /v
```

Displays detailed information about scheduled tasks.

**PowerShell**

```powershell
Get-ScheduledTask
```

Lists registered scheduled tasks.

```powershell
Get-ScheduledTask | Where-Object {$_.State -eq "Ready"}
```

Displays tasks in the Ready state.

## 4.2. Inspecting Task Details

```powershell
Get-ScheduledTask | Select-Object TaskName, TaskPath, State
```

Inspect a specific task:

```powershell
Get-ScheduledTask -TaskName "<TASK_NAME>" |
Format-List *
```

Display task execution information:

```cmd
schtasks /query /tn "<TASK_NAME>" /fo LIST /v
```

**What to Look For**

- Tasks running as SYSTEM.
- Tasks running as administrator accounts.
- Scripts executed automatically.
- Executables stored in writable directories.
- Tasks with weak permissions.
- Sensitive information stored in scripts.

## 4.3. Inspecting Scheduled Task Actions

A scheduled task may execute a PowerShell script, batch file, or executable.

Example:

```text
Action:
C:\Scripts\backup.ps1

Run As:
NT AUTHORITY\SYSTEM
```

If an ordinary user can modify `backup.ps1`, they may be able to influence code executed by the privileged task.

Check file permissions:

```cmd
icacls C:\Scripts\backup.ps1
```

**Important:** The task must actually execute the modified file, and the user must have the required access. A SYSTEM task alone does not establish a vulnerability.

---

# 5. File and Directory Permissions

Windows uses Access Control Lists (ACLs) to determine which users and groups can access files and directories.

During privilege escalation, inspecting file permissions is important because privileged processes may execute files that ordinary users can modify.

## 5.1. Understanding Windows Permissions

| Permission | Description |
|---|---|
| Read | Allows reading file contents. |
| Write | Allows modifying file contents. |
| Modify | Allows reading, writing, and deleting. |
| Read & Execute | Allows reading and executing files. |
| Full Control | Provides extensive access, including permission changes. |
| Write Data | Allows writing data to a file. |
| Delete | Allows deleting the object. |

## 5.2. Inspecting File Permissions

```cmd
icacls C:\Path\To\File
```

Example:

```cmd
icacls C:\Scripts\backup.ps1
```

Inspect directory permissions:

```cmd
icacls C:\Scripts
```

## 5.3. Finding Interesting Files

Search for common configuration files:

```cmd
where /r C:\ *.config
```

```cmd
where /r C:\ *.ini
```

```cmd
where /r C:\ *.xml
```

Search for files containing password references:

```cmd
findstr /si password *.xml *.ini *.config *.txt
```

**What to Look For**

- Writable service executables.
- Writable scheduled task scripts.
- Configuration files containing credentials.
- Backup files.
- Scripts executed with elevated privileges.
- Sensitive application data.

## 5.4. Important Windows Directories

```text
C:\Windows\System32\
C:\Windows\Temp\
C:\Windows\Tasks\
C:\ProgramData\
C:\Program Files\
C:\Users\
C:\Users\<USERNAME>\AppData\
C:\inetpub\wwwroot\
```

These locations may contain useful system or application information.

Their permissions vary depending on the system configuration.

**Key Takeaway:** A file becomes interesting when its permissions allow a lower-privileged user to influence something that executes with higher privileges.

---

# 6. Registry Enumeration

The Windows Registry is a hierarchical database that stores operating system, application, hardware, and user configuration information.

Registry entries can contain:

- Service configurations.
- Application settings.
- Startup information.
- User preferences.
- Authentication-related configuration.
- System security policies.

Incorrect registry permissions or exposed configuration values may create security weaknesses.

## 6.1. Registry Structure

| Registry Hive | Description |
|---|---|
| HKEY_LOCAL_MACHINE (HKLM) | Machine-wide configuration. |
| HKEY_CURRENT_USER (HKCU) | Current user's configuration. |
| HKEY_CLASSES_ROOT (HKCR) | File associations and COM registration. |
| HKEY_USERS (HKU) | Loaded user profiles. |
| HKEY_CURRENT_CONFIG (HKCC) | Current hardware profile. |

## 6.2. Enumerating Registry Keys

**CMD**

```cmd
reg query HKLM\SOFTWARE
```

```cmd
reg query HKCU\SOFTWARE
```

Query a specific key:

```cmd
reg query "<REGISTRY_PATH>"
```

**PowerShell**

```powershell
Get-ChildItem HKLM:\SOFTWARE
```

```powershell
Get-ChildItem HKCU:\SOFTWARE
```

## 6.3. AutoLogon Configuration

Windows can be configured to automatically log in using stored account information.

Inspect the Winlogon registry key:

```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

Important values include:

```text
DefaultUserName
DefaultPassword
AutoAdminLogon
DefaultDomainName
```

**What to Look For**

- Exposed credentials.
- Insecure application configuration.
- Registry keys writable by ordinary users.
- Startup-related configuration.
- Service-related registry settings.

**Important:** Registry values containing credentials should be handled as sensitive information. Never publish actual passwords in reports or repositories.

---

# 7. Credential Hunting

Credentials are sometimes stored in configuration files, scripts, application directories, and Windows deployment files.

Finding exposed credentials may provide access to additional services or accounts.

## 7.1. Important Locations

```text
C:\Windows\Panther\
C:\Windows\System32\Sysprep\
C:\ProgramData\
C:\inetpub\wwwroot\
C:\Users\<USERNAME>\AppData\Roaming\
C:\Users\<USERNAME>\Desktop\
C:\Users\<USERNAME>\Documents\
```

## 7.2. Common Files

```text
unattend.xml
sysprep.inf
web.config
groups.xml
app.config
settings.ini
```

These files may contain application settings or deployment-related information.

## 7.3. Search for Password References

```cmd
findstr /si password *.txt *.ini *.xml *.config
```

Search for configuration files:

```cmd
where /r C:\ *.config
```

Search for XML files:

```cmd
where /r C:\ *.xml
```

## 7.4. Windows Credential Manager

Windows Credential Manager stores credentials used by applications and network services.

Enumerate stored credentials:

```cmd
cmdkey /list
```

This displays saved credential targets and associated usernames where available.

**What to Look For**

- Stored credentials.
- Deployment configuration files.
- Application passwords.
- Scripts containing authentication information.
- Credentials associated with network resources.

**Important:** Never assume that a discovered username and password can be reused everywhere. Authentication and authorization differ between services.

---

# 8. Token Impersonation

Windows Access Tokens define the security context of processes and threads.

Token impersonation is an important Windows security mechanism that allows a thread to temporarily operate using another security identity.

Certain privileges may create opportunities for privilege escalation when combined with vulnerable service configurations.

## 8.1. Important Privileges

| Privilege | Description |
|---|---|
| SeImpersonatePrivilege | Allows impersonating a client under applicable conditions. |
| SeAssignPrimaryTokenPrivilege | Allows assigning primary process tokens under applicable conditions. |
| SeDebugPrivilege | Provides extensive debugging access to processes. |
| SeBackupPrivilege | Allows certain backup operations to bypass file access checks. |
| SeRestorePrivilege | Allows certain restore operations to bypass file access checks. |
| SeTakeOwnershipPrivilege | Allows taking ownership of securable objects. |

## 8.2. Enumeration

```cmd
whoami /priv
```

```cmd
whoami /all
```

## 8.3. Important Techniques to Study

- JuicyPotato
- RoguePotato
- PrintSpoofer
- GodPotato
- SweetPotato

These techniques are dependent on Windows versions, privileges, service configurations, and available mitigations.

**Key Takeaway:** Token impersonation is not simply about finding a privilege in `whoami /priv`. You must understand how Windows authentication, access tokens, service accounts, and impersonation levels interact.

---

# 9. Automated Enumeration Tools

Manual enumeration is important for understanding Windows internals, but automated tools can help identify potential misconfigurations more efficiently.

## 9.1. WinPEAS

WinPEAS is a Windows privilege escalation enumeration tool.

It checks different areas of the operating system for potentially interesting security configurations.

Repository:

https://github.com/peass-ng/PEASS-ng

## 9.2. Seatbelt

Seatbelt is a Windows security enumeration tool developed by SpecterOps.

It collects information about the operating system, users, processes, services, and other security-related configurations.

Repository:

https://github.com/GhostPack/Seatbelt

## 9.3. PowerUp

PowerUp is a PowerShell-based Windows privilege escalation assessment tool.

It checks for common Windows privilege escalation misconfigurations.

Repository:

https://github.com/PowerShellMafia/PowerSploit

## 9.4. AccessChk

AccessChk is a Microsoft Sysinternals utility used to inspect effective permissions on Windows objects.

It can help investigate:

- Files and directories.
- Registry keys.
- Services.
- Named pipes.
- Other securable objects.

Repository:

https://learn.microsoft.com/en-us/sysinternals/downloads/accesschk

## 9.5. PrivescCheck

PrivescCheck is a PowerShell script that performs Windows privilege escalation checks.

Repository:

https://github.com/itm4n/PrivescCheck

**Important:** Automated tools produce findings that require manual verification. A reported misconfiguration is not necessarily exploitable.

---

# 10. Privilege Escalation Workflow

A systematic workflow helps avoid missing important information during a Windows security assessment.

## Step 1 — Initial Enumeration

Identify the operating system, current user, groups, privileges, installed software, and network configuration.

```cmd
systeminfo
whoami /all
hostname
ipconfig /all
```

## Step 2 — Process Enumeration

Identify running processes and their execution contexts.

```cmd
tasklist /v
tasklist /svc
```

## Step 3 — Service Enumeration

Inspect service accounts, executable paths, and permissions.

```cmd
sc query state= all
```

```powershell
Get-CimInstance Win32_Service |
Select-Object Name, State, StartName, PathName
```

## Step 4 — Scheduled Tasks

Inspect tasks running with elevated privileges.

```cmd
schtasks /query /fo LIST /v
```

## Step 5 — File and Registry Permissions

Inspect potentially writable files, directories, and registry keys.

```cmd
icacls C:\Path\To\File
```

```cmd
reg query HKLM\SOFTWARE
```

## Step 6 — Credential Hunting

Inspect accessible configuration files, stored credentials, and application settings.

```cmd
cmdkey /list
```

## Step 7 — Security Privileges

Review the current token and identify potentially relevant privileges.

```cmd
whoami /priv
```

## Step 8 — Automated Enumeration

Use tools such as WinPEAS, Seatbelt, and PrivescCheck to identify additional potential attack vectors.

## Step 9 — Validate Findings

Investigate each suspected vulnerability manually.

Determine:
- What configuration is incorrect?
- Which permissions are involved?
- Which account executes the affected component?
- What conditions are required for exploitation?
- What is the potential security impact?

## Step 10 — Document and Remediate

Document the vulnerability, evidence, impact, and recommended remediation.

---

# Final Notes

Windows privilege escalation requires understanding how the operating system manages users, processes, services, permissions, access tokens, and inter-process communication.

The most important principle is to understand why a configuration is vulnerable rather than simply memorizing exploitation commands.

**Enumeration → Identification → Analysis → Validation → Privilege Escalation → Documentation**

All techniques should be tested only in authorized environments.
