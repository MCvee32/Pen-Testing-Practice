# AD Network Enumeration — Host & Service Discovery

## Scenario
VPN access to an Active Directory network, no credentials, target scope: `10.211.11.0/24`. Goal: discover live hosts, identify services, and map the AD environment.

## Host Discovery

### fping
Uses ICMP like `ping`, but supports scanning a whole subnet, moving to the next target after each probe rather than waiting on one:
```bash
fping -agq 10.211.11.0/24
```
- `-a` — show alive hosts only
- `-g` — generate target list from a subnet/netmask
- `-q` — quiet mode (suppress per-probe output/errors)

Example result: 4 live hosts found. Gateway and VPN server IPs are excluded (out of scope); remaining hosts saved to `hosts.txt`:
```bash
cat hosts.txt
10.211.11.20
10.211.11.10
```

### Nmap (Ping Scan)
Alternative for subnet-wide host discovery:
```bash
nmap -sn 10.211.11.0/24
```
- `-sn` — ping scan only, no port scanning

## Port Scanning

### Identifying the Domain Controller
Key AD-related ports/protocols to check:

| Port | Protocol | Significance |
|---|---|---|
| 88 | Kerberos | Kerberos-based enumeration |
| 135 | MS-RPC | RPC enumeration (null sessions) |
| 139 | SMB/NetBIOS | Legacy SMB access |
| 389 | LDAP | LDAP queries to AD |
| 445 | SMB | Modern SMB access, key for enumeration |
| 464 | Kerberos (kpasswd) | Password-related Kerberos service |

Targeted service/version scan:
```bash
nmap -p 88,135,139,389,445 -sV -sC -iL hosts.txt
```
- `-sV` — version detection
- `-sC` — run default NSE scripts
- `-iL hosts.txt` — read targets from file

**Identifying the DC:** typically has ports 88, 389, and 445 open, often with "Windows Server" banners or domain names visible in output.

### Full Port Scan (Exhaustive Assessment)
Useful when services might run on non-standard ports:
```bash
nmap -sS -p- -T3 -iL hosts.txt -oN full_port_scan.txt
```
- `-sS` — stealthy TCP SYN scan
- `-p-` — all 65,535 TCP ports
- `-T3` — "normal" timing (balances speed/stealth)
- `-iL hosts.txt` — input host list
- `-oN full_port_scan.txt` — save output to file

## Outcome
Identified 2 live in-scope hosts (1 Domain Controller, 1 Workstation) and the domain name, plus confirmed key services on the DC for further AD enumeration.

# SMB Share Enumeration

## Scenario
Breached perimeter, Linux attack box, no credentials yet. Goal: enumerate SMB shares, attempt anonymous access, and pull down accessible files.

## Discovering Services
Relevant Windows/AD ports to scan:

| Port | Service | Relevance |
|---|---|---|
| 88 | Kerberos | Ticket attacks (Pass-the-Ticket, Kerberoasting) |
| 135 | RPC Endpoint Mapper | Service discovery for lateral movement / DCOM RCE |
| 139 | NetBIOS Session Service | Null sessions, legacy file sharing |
| 389 | LDAP | Plaintext AD object/user/policy enumeration |
| 445 | SMB | File sharing, EternalBlue, SMB relay, credential theft |
| 636 | LDAPS | Encrypted LDAP; still exploitable via misconfig / AD CS attacks |

Scan command:
```bash
nmap -p 88,135,139,389,445,636 -sV -sC TARGET_IP
```
- `-sV` — version detection
- `-sC` — default NSE scripts

Open Kerberos + LDAP + SMB ports strongly suggest an AD environment, possibly a Domain Controller.

## Listing SMB Shares (Anonymous / Null Session)

### smbclient
Lists shares with no credentials:
```bash
smbclient -L //TARGET_IP -N
```
- `-L` — list shares
- `-N` — no password (null session)

### smbmap
Shows read/write permissions per share without manually connecting to each:
```bash
smbmap -H TARGET_IP
```
need to download.

### Nmap alternative
```bash
nmap -p445 --script smb-enum-shares TARGET_IP
```

**Non-standard shares worth investigating** typically include custom names like `AnonShare`, `SharedFiles`, `UserBackups` (as opposed to default admin shares like `ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `SYSVOL`).

## Accessing SMB Shares
Connect to a share with read access:
```bash
smbclient //TARGET_IP/SHARE_NAME -N
```
Inside the session:  
`ls`  
`get <filename>`  

With valid credentials instead of `-N`:
```bash
--user=USERNAME --password=PASSWORD
# or
-U 'username%password'
```
Use `-W` to specify the domain for domain accounts.

## What Can Be Found in SMB Shares
Anonymous shares are a misconfiguration risk, sometimes left for legacy device compatibility (old printers/scanners). May contain:
- Configuration files
- Backup files
- Scripts
- Documents with usernames/credentials (potentially unrotated passwords)

Writable shares also risk further abuse (e.g., uploading malicious files).

## Other Useful Tools
- **impacket-smbclient** — Python-based smbclient from the Impacket toolkit (`/opt/impacket/examples/` on AttackBox).
- **CrackMapExec** — enumeration + post-exploitation; includes SMB modules for share listing and credential testing.
- **enum4linux / enum4linux-ng** — extensive SMB enumeration:
```bash
  enum4linux -a TARGET_IP
```
  (worth redirecting output to a file for review)
- **Nmap `smb-enum-shares` script** — as covered above.

# Active Directory User Enumeration (Unauthenticated)

## LDAP Enumeration (Anonymous Bind)
LDAP provides a central directory for AD objects (users, groups, devices). Some servers allow anonymous read-only queries.

**Test for anonymous bind:**
```bash
ldapsearch -x -H ldap://TARGET_IP -s base
```
- `-x` — simple (anonymous) authentication
- `-H` — LDAP server URL
- `-s base` — limit query to the base object only

If enabled, returns domain metadata (naming contexts, DC hostname, functionality levels, etc.).

**Query user objects:**
```bash
ldapsearch -x -H ldap://TARGET_IP -b "dc=domain,dc=loc" "(objectClass=person)"
```

## Enum4linux-ng
Automates enumeration over SMB/RPC — users, groups, shares, password policy, etc.
```bash
enum4linux-ng -A TARGET_IP -oA results.txt
```
- `-A` — run all enumeration functions
- `-oA` — output to YAML and JSON

## RPC Enumeration (Null Sessions)
If SMB allows null sessions, unauthenticated access to `IPC$` can expose users/groups/shares via RPC.

**Verify null session access:**
```bash
rpcclient -U "" TARGET_IP -N
```
- `-U ""` — empty username (anonymous)
- `-N` — no password prompt

**Enumerate domain users:**
`rpcclient $> enumdomusers`
Returns usernames with their RIDs (e.g., `user:[gerald.burgess] rid:[0x650]`).

## RID Cycling
RIDs are part of a user/group's SID; certain values are standardized:
- `500` — Administrator
- `501` — Guest
- `512–514` — Domain Admins / Domain Users / Domain Guests
- User accounts generally start from `1000+`

If `enumdomusers` is restricted, brute-force individual RIDs manually:
```bash
for i in $(seq 500 2000); do echo "queryuser $i" | rpcclient -U "" -N TARGET_IP 2>/dev/null | grep -i "User Name"; done
```
- Loops through RID range, queries each, suppresses errors, filters for username lines.
- Can take 2–3 minutes to run.

## Username Enumeration With Kerbrute
Kerberos uses a ticket-based system via the KDC (vs. NTLM's challenge-response). Kerbrute brute-forces/enumerates valid AD usernames by abusing Kerberos pre-authentication.

**Why it matters:** Usernames from enum4linux-ng/rpcclient may include disabled accounts, non-domain accounts, honeypots, or false positives. Kerbrute confirms which are real, active AD users — useful for targeted password spraying.

**Installation:**
1. Download precompiled binary: https://github.com/ropnop/kerbrute/releases
2. Rename to `kerbrute`
3. `chmod +x kerbrute`

**Usage:**
```bash
./kerbrute userenum --dc TARGET_IP -d domain.loc users.txt
```
Outputs `VALID USERNAME` for each confirmed account and a summary count (e.g., "28 valid" of 33 tested).

**If other tools fail:** use a generic username wordlist with Kerbrute to discover accounts, then compile confirmed users into `users.txt` for password spraying.

## Summary
These techniques (LDAP anonymous bind, enum4linux-ng, RPC null sessions, RID cycling, Kerbrute) build a validated list of domain usernames — the foundation for a subsequent password spraying attack to obtain initial AD credentials.

# Password Spraying Attacks

## What It Is
Password spraying tests a small set of common passwords across many accounts (opposite of brute-force, which tests many passwords against one account). This avoids triggering account lockouts.

**Why it works:**
- Forced frequent password changes push users toward predictable patterns (e.g., `Summer2025!`).
- Weak enforcement of password policies.
- Reuse of common passwords across accounts.

**Common password sources:**
- Seasonal passwords
- IT default passwords (`Password123`)
- Breach-leaked password lists, e.g. `rockyou.txt`

## Step 1 — Enumerate the Password Policy
Must know minimum length, complexity, and lockout threshold before spraying (to avoid lockouts).

### rpcclient (null session)
```bash
rpcclient -U "" TARGET_IP -N
```
Then:  
`rpcclient $> getdompwinfo`  
example output:  
min_password_length: 12  
password_properties: 0x00000001  
DOMAIN_PASSWORD_COMPLEX  

### CrackMapExec
```bash
crackmapexec smb TARGET_IP --pass-pol
```
Returns detailed policy: min length, history length, max/min password age, complexity flags, lockout threshold, lockout duration, reset counter, etc.

**Complexity flag `0x00000001` / `000001`** means at least 3 of 4 conditions required:
- Uppercase letters
- Lowercase letters
- Digits
- Special characters

Also: passwords can't contain the account name or >2 consecutive characters of the user's full name.

## Step 2 — Build a Policy-Compliant Password List from rockyou.txt
Spraying the entire `rockyou.txt` against live AD accounts is **not viable** — it's enormous and would blow past the lockout threshold almost immediately. Instead, filter it down to a short list that satisfies the discovered policy (min length, complexity) before spraying.

Example filter for a policy requiring 12+ chars, complexity (upper/lower/digit/special), using `grep`/`pcregrep`:
```bash
# Minimum length (adjust to policy, e.g. 12)
grep -E '^.{12,}$' rockyou.txt > len_filtered.txt

# Require upper, lower, digit, and special char (adjust to taste)
pcregrep '^(?=.*[A-Z])(?=.*[a-z])(?=.*[0-9])(?=.*[^A-Za-z0-9]).*$' len_filtered.txt > policy_filtered.txt

# Trim to a small, lockout-safe batch — stay under the account lockout threshold
head -n 5 policy_filtered.txt > passwords.txt
```
Cross-reference with OSINT (data breach mentions, seasonal patterns, company name variants) to prioritize likely candidates rather than spraying blindly.

**Rule of thumb:** keep your password list smaller than the account lockout threshold (e.g., under 10 attempts per account per lockout-reset window), and space out spray attempts across that window if you need to try more passwords.

## Step 3 — Run the Spray
```bash
crackmapexec smb TARGET_IP -u users.txt -p passwords.txt
```
Example output:  
SMB TARGET_IP 445 WRK [-] tryhackme.loc\Administrator:Password! STATUS_LOGON_FAILURE  
SMB TARGET_IP 445 WRK [-] tryhackme.loc\Guest:Password! STATUS_LOGON_FAILURE  
...  
SMB TARGET_IP 445 WRK [+] tryhackme.loc***:****  
- `[-]` = failed login (`STATUS_LOGON_FAILURE`)
- `[+]` = **valid credential pair found**

## Key Takeaway
Password policy enumeration isn't just recon — it directly shapes which passwords from a large list like `rockyou.txt` are even worth trying, and how many attempts you can safely make per account before risking lockout.
