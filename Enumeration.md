# Windows Privilege Escalation: Initial Enumeration

Initial information gathering after gaining access to a Windows host to identify potential privilege escalation vectors.

---

## Key Data Points

* **OS Name:** Knowing the type of Windows OS (workstation or server) and level (Windows 7 or 10, Server 2008, 2012, 2016, 2019, etc.) gives an idea of the types of tools available (such as the PowerShell version), or lack thereof on legacy systems, and helps identify public exploits for that OS version.
* **Version:** There may be public exploits targeting a vulnerability in a specific version of Windows.
  > **Warning:** Windows system exploits can cause system instability or a complete crash. Be careful running these against any production system, and make sure you fully understand the exploit and possible ramifications before running one.
* **Running Services:** Knowing what services are running on the host is important, especially those running as `NT AUTHORITY\SYSTEM` or an administrator-level account. A misconfigured or vulnerable service running in the context of a privileged account can be an easy win for privilege escalation.

---

## System Information & Patch Level

Gather basic system details, environment variables, installed patches, and software.

```cmd
:: Look at running processes and associated services
tasklist /svc

:: Display all environment variables
set

:: Check OS version, architecture, virtualization status, and recent patches
systeminfo
```

### Checking Hotfixes & Installed Software
If `systeminfo` does not display hotfixes, query them via WMI or PowerShell:

```cmd
:: Display patches using WMI with QFE (Quick Fix Engineering)
wmic qfe

:: Display installed software via WMI
wmic product get name
```

```powershell
# Display patches via PowerShell
Get-HotFix | ft -AutoSize

# Display installed software and versions via PowerShell
Get-WmiObject -Class Win32_Product | select Name, Version
```

---

## Network Connections

Display active TCP and UDP connections to see which services are listening on which ports (both locally and externally):

```cmd
netstat -ano
```

---

## User & Group Enumeration

Determine the current user context, privileges, logged-in users, and local group memberships.

### Current User & Privileges
> **Note:** Always check what user context your account is running under first—you may already be `SYSTEM` or equivalent. If running as a service account, you may have privileges such as `SeImpersonatePrivilege`, which can often be abused using tools like Juicy Potato.

```cmd
:: Check current username
echo %USERNAME%

:: Check group memberships and inherited rights
whoami /groups
```

### Logged-In Users & Local Accounts
Check other users on the system (valuable files such as scripts with passwords or SSH keys are often found in a user's `Desktop`, `Documents`, or `Downloads` folder):

```cmd
:: Determine which users are logged into the system (idle or active)
query user

:: List local user accounts
net user

:: List local groups
net localgroup
```

### Group Details & Password Policy
> **Note:** Check details for any non-standard groups—you may find a password or other useful information stored in the group's description, or discover credentials of another non-admin user who is a member of a privileged local group.

```cmd
:: View members and details of a specific group
net localgroup administrators

:: Get password policy and other account information
net accounts
```
