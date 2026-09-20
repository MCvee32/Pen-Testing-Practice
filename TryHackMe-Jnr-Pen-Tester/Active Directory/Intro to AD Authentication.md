## NTLM Authentication

### What is NTLM?
NetNTLM (NTLM) is a challenge-response authentication protocol dating back 
to early Windows NT. Largely superseded by Kerberos as the AD default, but 
still used in legacy systems, workgroups, and as a fallback.

**Versions:**
- **NTLMv1** — original, highly insecure
- **NTLMv2** — stronger cryptography, but still vulnerable to various attacks

### How It Differs from Kerberos
- **NTLM**: client authenticates directly *to the service*, which then 
  verifies identity against the domain controller
- **Kerberos**: client authenticates *to the domain controller first* and 
  presents a ticket to services

### Authentication Flow
1. Client sends username to the server to request access to a service
2. Server generates a random 16-byte **challenge (nonce)** and sends it 
   to the client
3. Client encrypts the challenge using the **NT hash** of their password, 
   sends the response back
4. Server forwards username, challenge, and response to the **domain 
   controller (DC)**
5. DC retrieves the user's stored NT hash and encrypts the same challenge
6. DC compares its result to the client's response
7. Server grants/denies access based on the DC's verdict

> The password itself is never transmitted — only the encrypted challenge 
> response (a "zero-knowledge proof").

### Benefits
- Simple to implement — no KDC infrastructure needed
- No clock synchronization required (unlike Kerberos)
- Reliable fallback when Kerberos fails
- Works in non-domain workgroup environments

### Drawbacks / Attack Surface
- **No mutual authentication** — client can't verify the server → MITM risk
- **Weak cryptography** — NTLMv1 uses DES + unsalted hashes (crackable); 
  NTLMv2 stores unsalted hashes in memory
- **Relay attacks** — attacker intercepts and relays NTLM auth to another service
- **Pass-the-hash** — NT hash alone is enough to authenticate; no need 
  to crack the password
- **Slower** — every auth requires a round trip to the DC

### When NTLM Still Gets Used
- DC is unreachable for Kerberos
- Accessing a resource by **IP address** (Kerberos needs an SPN/DNS name)
- Target service has no registered **SPN** in AD
- Authenticating to non-domain-joined systems
- Legacy apps that require it

## Kerberos Authentication

### What is Kerberos?
Ticket-based network authentication protocol developed by MIT, adopted by 
Microsoft as the AD default starting with Windows 2000. Authenticates via 
a trusted third party — the **Key Distribution Center (KDC)**.

**Key difference from NTLM:** with Kerberos, the client authenticates to 
the domain controller *first* and receives tickets to present to services 
(reverse of NTLM's flow).

### Key Components

| Component | Description |
|---|---|
| **KDC** | Service on the DC handling ticket requests; made up of AS + TGS |
| **Authentication Service (AS)** | Verifies identity, issues the initial TGT |
| **Ticket Granting Service (TGS)** | Issues service tickets to holders of a valid TGT |
| **TGT** | Initial "primary ticket" issued after successful auth |
| **Service Ticket (ST)** | Grants access to a specific service; obtained via TGT |
| **SPN** | Unique identifier tying a service instance to an account |
| **KRBTGT account** | Its password hash encrypts all TGTs; compromise → Golden Tickets |

### Authentication Flow (5 steps, 8 processes)

**1–2. AS-REQ / AS-REP**
- Client sends username + timestamp encrypted with the user's password 
  hash (pre-authentication) to the KDC
- KDC verifies by decrypting the timestamp, then replies with:
  - a session key (encrypted with the user's password hash)
  - a **TGT** (encrypted with the KRBTGT hash — client can't read/modify it)

**3–4. TGS-REQ / TGS-REP**
- Client sends the TGT + target SPN + an authenticator (encrypted with 
  the session key) to request a service ticket
- KDC decrypts the TGT, validates, and returns:
  - a **Service Ticket** (encrypted with the target service's password hash)
  - a service session key (encrypted with the original session key)

**5. AP-REQ**
- Client presents the Service Ticket to the target service
- Service decrypts it with its own password hash, validates identity, 
  grants access

### Benefits
- **Mutual authentication** — both sides verify each other (MITM-resistant)
- **No password transmission** — only encrypted tickets/session keys travel
- **Single Sign-On (SSO)** — one TGT covers multiple services
- **Delegation support** — services can act on a user's behalf
- **Better performance** — DC contacted only for initial auth / new tickets; 
  services validate locally afterward
- **Time-limited tickets** — TGTs typically valid ~10 hours

### Drawbacks / Attack Surface
- **Requires time sync** — clocks must be within 5 minutes or auth fails
- **Single point of failure** — KDC down = Kerberos auth fails (NTLM fallback possible)
- **Pass-the-Ticket** — stolen tickets let an attacker impersonate a user
- **Golden Tickets** — forged TGTs via a compromised KRBTGT hash
- **Kerberoasting** — service tickets (encrypted with service account 
  hashes) can be requested by any authenticated user and cracked offline
- **Deployment complexity** — needs correct SPNs, DNS, and time sync

### Credential Cache (ccache) Files — Linux
- Store a user's TGT and service tickets for the session
- Default path: `/tmp/krb5cc_%{uid}` (e.g. `/tmp/krb5cc_1000`)
- `KRB5CCNAME` env var selects the active ccache file
- `klist` displays tickets in the current ccache
- Tools like Impacket can auth using a ccache file with no password
- **Attack relevance:** stealing a user's ccache enables **Pass-the-Ticket / 
  Pass-the-ccache** — authenticating as them without their password

## AD Authentication Weaknesses — Attack Overview

Both NTLM and Kerberos have well-known, decades-old weaknesses that remain 
widely exploited in real-world AD compromises. Credential-based attacks 
account for the majority of successful breaches; Microsoft has announced 
plans to eventually deprecate NTLM, though the transition is still years out.

### NTLM-Specific Weaknesses
- **Weak Cryptography** — unsalted MD4 hashing → vulnerable to rainbow 
  tables and fast GPU brute-forcing
- **Pass-the-Hash (PtH)** — the NT hash itself is usable for authentication, 
  so stealing it skips the need for the plaintext password
- **NTLM Relay** — no mutual authentication lets attackers intercept and 
  relay auth attempts to other services
- **Downgrade Attacks** — forcing a fallback from Kerberos to NTLM to 
  expose the weaker protocol
- **No Mutual Authentication** — server identity isn't verified, easing MITM

### Kerberos-Specific Weaknesses
- **Kerberoasting** — any authenticated user can request service tickets 
  (SPN-encrypted with the service account hash) and crack them offline
- **AS-REP Roasting** — accounts with Kerberos pre-auth disabled leak 
  crackable password hashes with no prior authentication needed
- **Pass-the-Ticket (PtT)** — tickets extracted from memory can be replayed 
  to impersonate their owner
- **Overpass-the-Hash** — an NTLM hash is used to request a Kerberos TGT, 
  converting an NTLM compromise into a Kerberos one
- **Golden Ticket** — a compromised KRBTGT hash lets an attacker forge 
  TGTs for *any* domain user (including Domain Admins) → full, persistent 
  domain control
- **Silver Ticket** — like a Golden Ticket, but forged with a service 
  account's hash to mint service tickets for one resource, without 
  contacting the KDC

### Configuration-Based Weaknesses
- **Weak Passwords** — still the most common entry point overall
- **Password Spraying** — a few common passwords tried across many 
  accounts, often evading lockout thresholds
- **Misconfigured Delegation** — improper constrained/unconstrained 
  Kerberos delegation enabling privilege escalation and lateral movement
- **Stale Credentials** — old/unused service, former-employee, or machine 
  accounts with weak, unrotated passwords

### Takeaway
| Category | Key risk |
|---|---|
| NTLM | Hash *is* the credential — capture it, use it |
| Kerberos | Tickets *are* the credential — steal/forge them |
| Config | Human and admin error create the easiest openings |
