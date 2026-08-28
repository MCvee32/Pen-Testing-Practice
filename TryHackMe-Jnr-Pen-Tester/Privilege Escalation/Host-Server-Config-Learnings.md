## Host Server Configuration Reviews
# Configuration Review
Audit of a host's settings, permissions, services, and policies measured against an accepted secure baseline.  
From an offensive perspective this can reveal potential points of exploit.

# Offensive Enumeration Tools
Tools for privilege escalation enumeration
LinPEAS, WinPEAS and PowerUp are designed to run on compromised hosts and identify configuration weaknesses.

# Categories of Misconfiguration
- Users and Groups: users and groups violating POLP.
- File and Directory Permissions: Misconfigured permissions on sensitive files, system binaries, and configuration directories.
- Service Configurations: services that run as root or LocalSystem when they do not need to, service configuration binaries that are writable by non-privileged users, and service binary paths that are unquoted and contain spaces.
- Scheduled Tasks and Cron: Scheduled tasks that run with elevated privileges and reference scripts or binaries that a non-privileged user can modify are a direct escalation vector.
- Credential Storage: storing credentials in locations accessible to other accounts on the system.
- Network Configuration: Not associated with privilege escalation but important for a secure posture of a host.

# Structured Enumeration Methodology
Phase 1: Situational Awareness  
Understanding the context of the host/target. Shaping the enumeration that follows.  
Phase 2: Category-Based Enumeration  
Enumerating through each configuration category.  
Phase 3: Prioritisation and Exploitation  
Choosing the most direct/easiest path for privilege escalation
