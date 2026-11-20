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

## 3. Running Processes and Services

Enumerating running processes and services is important because some services operate under highly privileged accounts.

A vulnerable or misconfigured service running as `NT AUTHORITY\SYSTEM` or an administrator-level account may create a privilege escalation opportunity.

### 3.1. Enumerate Running Processes

**CMD**

```cmd
tasklist
```

Displays currently running processes.

```cmd
tasklist /svc
```

Displays running processes together with the services hosted by each process.

### 3.2. Enumerate Windows Services

**CMD**

```cmd
sc query
```

Displays information about Windows services and their current states.

**PowerShell**

```powershell
Get-Service
```

Lists installed services and their current status.

```powershell
Get-CimInstance Win32_Service | Select-Object Name, DisplayName, State, StartName, PathName
```

Displays service names, states, execution accounts, and executable paths.

### What to Look For

- Services running as `NT AUTHORITY\SYSTEM`.
- Services running under administrator accounts.
- Unusual or third-party services.
- Services with suspicious executable paths.
- Services with potentially weak file or directory permissions.
- Services with unquoted executable paths.
- Misconfigured service permissions.

> **Important:** A service running as SYSTEM is not automatically vulnerable. Investigate its configuration, executable permissions, service permissions, and associated files before identifying a privilege escalation vector.

---

## 4. Installed Software and Patches

Installed applications and missing security updates can reveal potential attack surfaces.

An outdated application may contain known vulnerabilities, while missing Windows security patches may expose the system to publicly documented vulnerabilities.

### 4.1. Enumerate Installed Patches

**CMD**

```cmd
wmic qfe
```

Displays installed Windows updates using WMIC on systems where the utility is available.

**PowerShell**

```powershell
Get-HotFix | Format-Table -AutoSize
```

Displays installed Windows hotfixes and their installation dates.

Alternatively:

```powershell
Get-CimInstance Win32_QuickFixEngineering
```

Retrieves installed hotfix information through CIM.

### 4.2. Enumerate Installed Software

**CMD**

```cmd
wmic product get name
```

Lists software registered through the Windows Installer provider, where WMIC is available.

**PowerShell**

```powershell
Get-ItemProperty HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\* |
Select-Object DisplayName, DisplayVersion
```

Lists applications registered in the 64-bit machine-wide uninstall registry location.

To inspect the 32-bit application registry location:

```powershell
Get-ItemProperty HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\* |
Select-Object DisplayName, DisplayVersion
```

### What to Look For

- Missing security updates.
- Outdated applications.
- Vulnerable software versions.
- Applications installed in unusual directories.
- Software running with elevated privileges.
- Known vulnerabilities associated with installed software.

> **Note:** WMIC is deprecated on modern Windows systems and may not be available. Also, `Win32_Product` should generally be avoided for routine software enumeration because querying it can trigger Windows Installer consistency checks.

---

## 5. Network Enumeration

Network enumeration helps identify the host's network configuration, active connections, neighboring devices, and potential access to additional internal networks.

This information can reveal network segmentation, VPN interfaces, internal infrastructure, and services communicating with the target machine.

### 5.1. Network Connections

**CMD**

```cmd
netstat -ano
```

Displays active TCP connections, listening ports, and associated process IDs.

To display only listening connections:

```cmd
netstat -ano | findstr LISTENING
```

To identify the process associated with a specific PID:

```cmd
tasklist /FI "PID eq 1234"
```

Replace `1234` with the actual process ID.

### What to Look For

- Locally listening ports.
- Connections to internal IP addresses.
- Unusual outbound connections.
- Services listening on non-standard ports.
- Processes associated with network connections.

### 5.2. ARP Table Enumeration

The Address Resolution Protocol (ARP) table maps IPv4 addresses to MAC addresses within the local network segment.

Inspecting this table can help identify previously discovered network neighbors and default gateways without generating additional network scan traffic.

**CMD**

```cmd
arp -a
```

### What to Look For

- **Network Neighbors:** Previously resolved local IP addresses and MAC addresses.
- **Default Gateway:** The device responsible for forwarding traffic outside the local subnet.
- **Dynamic Entries:** Automatically learned IP-to-MAC mappings.
- **Static Entries:** Manually configured ARP mappings.
- **Additional Subnets:** Potential clues about the surrounding network infrastructure.

> **Note:** ARP entries do not necessarily indicate that a device is currently online. They may represent previously resolved neighbors.

### 5.3. Routing Table Enumeration

The routing table determines how network traffic is forwarded to different destinations.

Reviewing routing information can reveal additional network interfaces, internal subnets, VPN connections, and gateways leading to segmented environments.

**CMD**

```cmd
route print
```

### What to Look For

- **Non-default Routes:** Destinations outside the local subnet.
- **Additional Gateways:** Routers providing access to other networks.
- **Interface Mappings:** IP addresses associated with network interfaces.
- **Internal Subnets:** Routes pointing to private network ranges.
- **VPN Routes:** Routes associated with VPN or tunneling interfaces.

---

## 6. Windows Defender Status

Windows Defender is an important endpoint security component that can detect suspicious files, scripts, and process behavior.

Checking its configuration helps determine which security mechanisms are enabled on the target system.

### Command

**PowerShell**

```powershell
Get-MpComputerStatus
```

### Important Properties

| Property                        | Description                                           |
| ------------------------------- | ----------------------------------------------------- |
| `AntivirusEnabled`              | Indicates whether antivirus protection is enabled.    |
| `RealTimeProtectionEnabled`     | Indicates whether real-time protection is enabled.    |
| `BehaviorMonitorEnabled`        | Indicates whether behavioral monitoring is enabled.   |
| `AntivirusSignatureLastUpdated` | Displays the last antivirus signature update time.    |
| `AMServiceEnabled`              | Indicates whether the antimalware service is enabled. |
| `IsTamperProtected`             | Indicates whether tamper protection is enabled.       |

### What to Look For

- Whether antivirus protection is enabled.
- Whether real-time protection is active.
- Whether behavioral monitoring is enabled.
- Whether antivirus signatures are up to date.
- Whether tamper protection is enabled.

> **Note:** The availability and output of Defender cmdlets depend on the Windows version, Defender installation, and security configuration.

---

## 7. AppLocker Policy Enumeration

AppLocker is a Windows security feature that controls which applications, scripts, installers, and other executable files users are allowed to run.

During privilege escalation assessments, AppLocker policies can reveal execution restrictions that may affect the use of enumeration scripts and security assessment tools.

### 7.1. Enumerate Effective AppLocker Rules

**PowerShell**

```powershell
Get-AppLockerPolicy -Effective | Select-Object -ExpandProperty RuleCollections
```

**Explanation:**

- `Get-AppLockerPolicy` retrieves AppLocker policy information.
- `-Effective` retrieves the effective policy applied to the computer.
- `Select-Object -ExpandProperty RuleCollections` displays the individual rule collections.

### 7.2. Test Local AppLocker Policy Against a Binary

**PowerShell**

```powershell
Get-AppLockerPolicy -Local | Test-AppLockerPolicy -Path C:\Windows\System32\cmd.exe -User Everyone
```

**Explanation:**

- `Get-AppLockerPolicy -Local` retrieves the local AppLocker policy.
- `Test-AppLockerPolicy` evaluates whether a specified file would be allowed or blocked.
- `-Path` specifies the executable to test.
- `-User Everyone` evaluates the policy for the Everyone group.

### What to Look For

- **Enforcement Mode:** Determine whether rule collections are enforced or configured for audit-only mode.
- **Allowed Execution Paths:** Identify paths covered by allow rules.
- **Restricted Locations:** Identify locations where execution may be blocked.
- **File Type Restrictions:** Check rules affecting executables, scripts, installers, and DLLs.
- **Publisher Rules:** Identify restrictions based on digital signatures and publishers.
- **Path Rules:** Determine whether execution permissions depend on file location.

> **Important:** Do not assume that directories such as `C:\Windows\Temp` or `C:\Windows\Tasks` are automatically allowed. Always inspect the actual policy and test the relevant execution context.

---

## Enumeration Checklist

Use this checklist during authorized Windows privilege escalation assessments.

### System Information

- [ ] Identify Windows version and build.
- [ ] Identify system architecture.
- [ ] Check PowerShell version.
- [ ] Review system information and installed patches.

### User and Group Enumeration

- [ ] Identify the current user.
- [ ] Enumerate user privileges.
- [ ] Inspect group memberships.
- [ ] Identify logged-in users.
- [ ] Enumerate local accounts and groups.
- [ ] Review local password policies.

### Processes and Services

- [ ] Enumerate running processes.
- [ ] Identify associated services.
- [ ] Identify services running as SYSTEM or administrator.
- [ ] Inspect service executable paths and permissions.

### Installed Software

- [ ] Enumerate installed applications.
- [ ] Identify application versions.
- [ ] Check installed Windows updates.
- [ ] Investigate potentially vulnerable software.

### Network Enumeration

- [ ] Enumerate active TCP and UDP connections.
- [ ] Identify listening ports and associated processes.
- [ ] Inspect the ARP table.
- [ ] Inspect the routing table.
- [ ] Identify additional interfaces and internal subnets.

### Security Mechanisms

- [ ] Check Windows Defender status.
- [ ] Review real-time protection and behavioral monitoring.
- [ ] Enumerate effective AppLocker policies.
- [ ] Identify application and script execution restrictions.

---

## Final Notes

Initial enumeration should be performed systematically before attempting privilege escalation.

The information collected during this stage helps identify potential attack surfaces, misconfigurations, excessive privileges, vulnerable services, and security restrictions.

**Remember:** Enumeration is not exploitation. Finding a suspicious configuration does not automatically mean that it is exploitable. Every potential vector requires further investigation and validation.

Process → Access Token → Inter-Process Communication (IPC)
# 2. Communication with Processes

One of the important areas to investigate during Windows privilege escalation is the communication between running processes.

Even if a process does not run with administrative privileges, it may provide an opportunity to obtain a higher level of access through its security context, exposed interfaces, or misconfigured communication mechanisms.

For example, a web server such as IIS or XAMPP may run under a dedicated service account. If an attacker obtains code execution through a vulnerable web application, the resulting shell will typically run under the identity of the web server process.

Although this account may not have administrative privileges, it could possess powerful Windows privileges such as `SeImpersonatePrivilege`.

Under certain conditions, this privilege can be abused through techniques associated with Rogue Potato, Juicy Potato, or PrintSpoofer to obtain `NT AUTHORITY\SYSTEM`.

**Important:** These techniques are version- and configuration-dependent. Having `SeImpersonatePrivilege` does not automatically guarantee SYSTEM access.

---

## Contents

1. [Access Tokens](#1-access-tokens)
2. [Network Connections and Local Services](#2-network-connections-and-local-services)
3. [Named and Anonymous Pipes](#3-named-and-anonymous-pipes)
   - [Understanding Named Pipes](#understanding-named-pipes)
   - [Enumerating Named Pipes](#enumerating-named-pipes)
   - [Inspecting Pipe Permissions](#inspecting-pipe-permissions)
4. [Key Takeaways](#4-key-takeaways)

---

## 1. Access Tokens

Windows Access Tokens define the security context of a process or thread.

An access token contains information about the identity of the user and the security permissions associated with that identity.

When a user successfully authenticates to Windows, the operating system creates an access token representing that user's security context.

When a process is created, it receives a primary access token. Threads can also use impersonation tokens to temporarily operate under another security context.

Windows uses these tokens to determine which resources a process or thread can access and which operations it is permitted to perform.

### Important Access Token Components

| Component | Description |
|---|---|
| User SID | Identifies the user account associated with the token. |
| Group SIDs | Identifies the security groups associated with the token. |
| Privileges | Defines special rights assigned to the token. |
| Integrity Level | Represents the process's integrity level, such as Low, Medium, High, or System. |
| Token Type | Indicates whether the token is primary or impersonation. |
| Restricted SIDs | Can limit the security access available to the token. |

### Primary Token vs. Impersonation Token

**Primary Token**

A primary token is normally associated with a process and defines the security context under which that process operates.

**Impersonation Token**

An impersonation token allows a thread to temporarily operate using another security identity.

This mechanism is commonly used by Windows services when handling requests from clients.

### Why Access Tokens Matter for Privilege Escalation

Some Windows privileges can provide opportunities to perform operations beyond the permissions normally associated with a user account.

For example:

| Privilege | Relevance |
|---|---|
| `SeImpersonatePrivilege` | May allow a process to impersonate another security identity under applicable conditions. |
| `SeAssignPrimaryTokenPrivilege` | May allow assigning a primary token to a process. |
| `SeDebugPrivilege` | Can provide extensive access to other processes. |
| `SeBackupPrivilege` | Can allow access to protected files through backup operations. |

### Enumeration Commands

**CMD**

```cmd
whoami /all
```

Displays the current user's security context, group memberships, and assigned privileges.

```cmd
whoami /priv
```

Displays the privileges assigned to the current access token.

**PowerShell**

```powershell
Get-Process -IncludeUserName
```

Displays running processes and their associated usernames where permissions allow.

### What to Look For

- Processes running under privileged service accounts.
- Processes running as `NT AUTHORITY\SYSTEM`.
- Accounts with `SeImpersonatePrivilege`.
- Accounts with `SeAssignPrimaryTokenPrivilege`.
- Differences between primary and impersonation tokens.

---

## 2. Network Connections and Local Services

Another useful approach is examining active network connections to identify services running on the target machine.

Some applications expose administrative interfaces or management services through local network sockets.

### Command

**CMD**

```cmd
netstat -ano
```

Displays active TCP connections, listening ports, and associated process IDs.

### Understanding Listening Addresses

Pay particular attention to the addresses on which services are listening.

| Address | Meaning |
|---|---|
| `127.0.0.1` | IPv4 loopback address. |
| `::1` | IPv6 loopback address. |
| `0.0.0.0` | Listening on all available IPv4 interfaces. |
| `::` | IPv6 wildcard address. |
| Specific IP | Listening on a particular network interface. |

### Why Loopback Services Are Interesting

Services listening on `127.0.0.1` or `::1` are generally accessible only from the local machine.

Administrators sometimes assume that local-only services are safe because they cannot be accessed directly from external network hosts.

However, a compromised local process may still be able to communicate with these services.

If a local service exposes an administrative interface without sufficient authentication or authorization, it may provide an additional attack surface.

### Example: FileZilla Server

In some FileZilla Server configurations, TCP port `14147` has been used for the administrative interface.

If this port appears in the network enumeration results, it may be worth investigating.

**Enumeration:**

```cmd
netstat -ano | findstr 14147
```

Identify the associated process:

```cmd
tasklist /FI "PID eq 1234"
```

Replace `1234` with the actual PID.

### What to Investigate

- Whether the service is listening on localhost or external interfaces.
- Whether authentication is required.
- Whether administrative access is restricted.
- Which user account runs the service.
- Whether the application contains misconfigurations.
- Whether the service allows privileged administrative operations.

**Important:** Finding an open local port does not automatically indicate a vulnerability. The actual application configuration and access controls must be investigated.

---

## 3. Named and Anonymous Pipes

Pipes are Windows Inter-Process Communication (IPC) mechanisms that allow processes to exchange data.

They are commonly used by Windows services and applications to communicate with other processes.

There are two main types:

- Named Pipes
- Anonymous Pipes

### Understanding Named Pipes

Named Pipes provide a communication channel between processes.

They can be used by processes running on the same computer or, depending on configuration, across a network.

A named pipe has a name that allows client processes to identify and connect to it.

Example:

```text
\\.\pipe\ExampleNamedPipe
```

Windows uses a client-server communication model for named pipes:

- **Server:** The process that creates and manages the named pipe.
- **Client:** The process that connects to the named pipe and communicates with the server.

### Communication Types

Named Pipes support two main communication modes.

**Half-Duplex**

A one-way communication channel in which data flows in one direction at a time.

**Full-Duplex**

A two-way communication channel that allows both the client and server to send and receive data.

A named pipe can also have multiple instances, allowing multiple clients to connect to separate instances associated with the same pipe name.

### Anonymous Pipes

Anonymous Pipes are another IPC mechanism used for communication between processes.

Unlike named pipes, anonymous pipes do not have a publicly addressable name.

They are commonly used for one-way communication, such as redirecting the output of one process into another process.

For example:

```cmd
ipconfig | findstr IPv4
```

In this example, the output of `ipconfig` is passed to `findstr` through a pipe.

---

### Enumerating Named Pipes

During privilege escalation assessments, identifying existing named pipes can reveal communication channels used by Windows services and applications.

**PowerShell**

```powershell
Get-ChildItem \\.\pipe\
```

Alternatively, using the shorter PowerShell alias:

```powershell
gci \\.\pipe\
```

Both commands enumerate named pipes accessible through the current security context.

**Sysinternals PipeList**

Microsoft Sysinternals provides a utility called PipeList that can enumerate named pipes.

```cmd
pipelist.exe /accepteula
```

This displays information about named pipes currently available on the system.

### What to Look For

- Pipes associated with privileged Windows services.
- Pipes created by third-party applications.
- Unusual or application-specific pipe names.
- Pipes that may expose administrative functionality.
- Communication channels with potentially weak access permissions.

---

### Inspecting Pipe Permissions

Finding a named pipe is only the first step.

The next step is determining which users and groups have permission to access it.

Windows uses security descriptors and Discretionary Access Control Lists (DACLs) to control access to named pipes.

A DACL contains Access Control Entries (ACEs) that specify which security principals are allowed or denied particular operations.

These permissions may include:

- Reading data.
- Writing data.
- Creating pipe instances.
- Connecting to a pipe.
- Modifying security settings.

### AccessChk

Microsoft Sysinternals provides AccessChk, a utility used to inspect effective permissions on Windows objects.

It can help identify whether a particular user or group has excessive permissions over a named pipe.

**Enumerate permissions for all named pipes:**

```cmd
accesschk.exe /accepteula \\.\Pipe\
```

**Inspect a specific named pipe:**

```cmd
accesschk.exe /accepteula -v \\.\Pipe\lsass
```

The `-v` option provides verbose information about the object's permissions.

### Understanding the Output

When reviewing AccessChk results, pay attention to:

| Permission | Description |
|---|---|
| Read | Allows reading permitted information from the object. |
| Write | Allows writing data where permitted. |
| Read/Write | Provides both reading and writing access. |
| Full Control | May allow extensive operations, depending on the object and granted rights. |

### What to Look For

- Permissions granted to `Everyone`.
- Permissions granted to `Authenticated Users`.
- Unexpected write access.
- Excessive permissions granted to ordinary users.
- Pipes associated with privileged services.
- Security configurations that allow unauthorized clients to communicate with sensitive services.

**Important:** Write access to a named pipe does not automatically provide privilege escalation. The server application's behavior, authentication requirements, and handling of client requests determine whether the permissions can be abused.

---

## 4. Key Takeaways

Process communication can expose additional attack surfaces during Windows privilege escalation assessments.

Understanding access tokens helps identify the security context of running processes, while examining network connections and named pipes can reveal interfaces through which processes communicate.

| Area | Main Objective |
|---|---|
| Access Tokens | Understand process and thread security contexts. |
| Primary Tokens | Identify the security identity assigned to processes. |
| Impersonation Tokens | Understand how threads can operate under another identity. |
| Network Connections | Identify local services and exposed interfaces. |
| Named Pipes | Discover IPC channels between processes. |
| Anonymous Pipes | Understand basic one-way process communication. |
| AccessChk | Inspect permissions assigned to Windows objects. |

**Key Takeaway:** A process does not necessarily need to run as Administrator to become a useful privilege escalation entry point. Its access token, privileges, exposed interfaces, and communication permissions may reveal additional opportunities.
