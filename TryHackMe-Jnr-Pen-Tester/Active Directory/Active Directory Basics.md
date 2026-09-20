## Room Overview
Introduces core AD concepts and components used to manage users, computers, 
and resources in Windows enterprise networks.

## Key Concepts Covered

### What is Active Directory (AD)?
- Microsoft directory service for centralized management of users, 
  computers, and network resources
- Provides authentication and authorization across a domain

### Core Components
- **Domain** — logical grouping of objects (users, computers) sharing 
  a common AD database
- **Domain Controller (DC)** — server hosting the AD database (NTDS.dit) 
  and handling authentication (Kerberos/NTLM)
- **Organizational Units (OUs)** — containers used to organize users, 
  groups, and computers, often mapped to Group Policy application
- **Users & Groups** — security principals; groups simplify permission 
  assignment (Security vs. Distribution groups)
- **Group Policy Objects (GPOs)** — centrally push configuration/security 
  settings to users and computers

### Trusts & Structure
- **Trees** — collections of domains sharing a contiguous namespace
- **Forests** — collections of trees; the top-level AD security boundary
- **Domain/Forest Trusts** — allow resource access across domains/forests 
  (one-way, two-way, transitive, etc.)

### Authentication Protocols (introduced conceptually)
- **Kerberos** — default AD authentication protocol (tickets: TGT/TGS)
- **NTLM** — legacy challenge-response protocol, still present for 
  backward compatibility
