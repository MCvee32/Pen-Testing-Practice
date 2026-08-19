## Passive Recon
Passive reconnaissance relies exclusively on publicly available information. No packets are sent to the target and no direct interaction occurs.   
- **WHOIS:** Domain registration details including registrar, dates, and name servers. Most personal details are now redacted for privacy.
- **DNS lookups:** A/AAAA (IP addresses), MX (mail servers), TXT (SPF/DMARC/verification), and other record types, queried via public resolvers like 1.1.1.1.
- **Subdomain enumeration:** DNSDumpster for DNS aggregation and graphing, and crt.sh for Certificate Transparency log searches, which is the most effective passive method for discovering subdomains via public SSL/TLS certificates.
- **Exposed services:** Shodan.io for device banners, ports, and hosting information.  
  
**Useful Commands:**
- **Lookup DNS A records**	`dig tryhackme.com A`
- **Lookup DNS MX records at specific server**	`dig @1.1.1.1 tryhackme.com MX`
- **Lookup DNS TXT records**	`dig tryhackme.com TXT`

## Active Recon
Active reconnaissance requires direct engagement with the target. Your probes can be logged, detected, or blocked.  
Gives five useful tools:
- `ping` confirms whether a target is reachable and provides TTL-based clues about its OS.
- `traceroute` to understand network path
- `telnet` and `netcat` to connect to individual ports to grab banners and identify running services with their versions (for HTTP-based services `curl` or `nc` is preferred over `telnet` for banner grabbing.

**Useful Commands:**
- `ping -c 10 MACHINE_IP`
- `traceroute MACHINE_IP`
- **traceroute IPv6**`traceroute -6 MACHINE_IPV6`
- `telnet MACHINE_IP PORT_NUMBER`
- **netcat as client:** `nc MACHINE_IP PORT_NUMBER`
- **netcat as server:**	`nc -lvnp PORT_NUMBER`

## Protocols and Servers
### Protocol and Servers Part 1
Looks at unencrypted protocols. 
**Protocol Reference Table**
| Protocol | TCP Port | Application(s)   | Data Security | Secure Alternative         | Secure Port                     |
|----------|----------|-------------------|----------------|------------------------------|----------------------------------|
| FTP      | 21       | File Transfer      | Cleartext      | FTPS or SFTP                 | 990 (FTPS), 22 (SFTP)           |
| HTTP     | 80       | Worldwide Web      | Cleartext      | HTTPS                        | 443                              |
| IMAP     | 143      | Email (MDA)        | Cleartext      | IMAPS                        | 993                              |
| POP3     | 110      | Email (MDA)        | Cleartext      | POP3S                        | 995                              |
| SMTP     | 25       | Email (MTA)        | Cleartext      | SMTPS or SMTP with STARTTLS  | 465 (SMTPS), 587 (Submission)   |
| Telnet   | 23       | Remote Access      | Cleartext      | SSH                          | 22                                |

### Protocols and Servers Part 2
Looks at three common network protocol attacks:
- Sniffing
- MITM
- Password  

Always use encrypted (secure) alternatives of protocols.
