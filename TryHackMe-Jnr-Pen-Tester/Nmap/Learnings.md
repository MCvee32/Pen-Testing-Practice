## Nmap Live Host Discovery
How to detect live hosts through ARP, ICMP, TCP, and UDP
**Nmap Commands:**  
| Scan Type                | Example Command                                  |
|---------------------------|---------------------------------------------------|
| ARP Scan                  | `sudo nmap -PR -sn 10.200.6.0/24`                 |
| ICMP Echo Scan            | `sudo nmap -PE -sn 10.200.6.0/24`                 |
| ICMP Timestamp Scan       | `sudo nmap -PP -sn 10.200.6.0/24`                 |
| ICMP Address Mask Scan    | `sudo nmap -PM -sn 10.200.6.0/24`                 |
| TCP SYN Ping Scan         | `sudo nmap -PS22,80,443 -sn 10.200.6.0/30`        |
| TCP ACK Ping Scan         | `sudo nmap -PA22,80,443 -sn 10.200.6.0/30`        |
| UDP Ping Scan             | `sudo nmap -PU53,161,162 -sn 10.200.6.0/30`       |
*Remember to add -sn if you are only interested in host discovery without port-scanning.  
Other options:  
-n: no DNS lookup
-R: for reverse-DNS lookup
-sn: host discovery only

## Nmap Basic Port Scans
| Port Scan Type   | Example Command               |
|-------------------|--------------------------------|
| TCP Connect Scan   | `nmap -sT 10.145.160.222`      |
| TCP SYN Scan       | `sudo nmap -sS 10.145.160.222` |
| UDP Scan           | `sudo nmap -sU 10.145.160.222` |

**Option flags:**  
| Option                  | Purpose                              |
|---------------------------|----------------------------------------|
| `-p-`                     | all ports                             |
| `-p1-1023`                | scan ports 1 to 1023                  |
| `-F`                      | 100 most common ports                 |
| `-r`                      | scan ports in consecutive order       |
| `-T<0-5>`                 | -T0 being the slowest and T5 the fastest |
| `--max-rate 50`           | rate <= 50 packets/sec                |
| `--min-rate 15`           | rate >= 15 packets/sec                |
| `--min-parallelism 100`   | at least 100 probes in parallel       |

## Nmap Advanced Port Scans
| Port Scan Type                | Example Command                                          |
|--------------------------------|-----------------------------------------------------------|
| TCP Null Scan                  | `sudo nmap -sN 10.145.182.254`                            |
| TCP FIN Scan                   | `sudo nmap -sF 10.145.182.254`                            |
| TCP Xmas Scan                  | `sudo nmap -sX 10.145.182.254`                            |
| TCP Maimon Scan                | `sudo nmap -sM 10.145.182.254`                            |
| TCP ACK Scan                   | `sudo nmap -sA 10.145.182.254`                            |
| TCP Window Scan                | `sudo nmap -sW 10.145.182.254`                            |
| Custom TCP Scan                | `sudo nmap --scanflags URGACKPSHRSTSYNFIN 10.145.182.254` |
| Spoofed Source IP              | `sudo nmap -S SPOOFED_IP 10.145.182.254`                  |
| Spoofed MAC Address            | `--spoof-mac SPOOFED_MAC`                                 |
| Decoy Scan                     | `nmap -D DECOY_IP,ME 10.145.182.254`                      |
| Idle (Zombie) Scan             | `sudo nmap -sI ZOMBIE_IP 10.145.182.254`                  |
| Fragment IP data into 8 bytes  | `-f`                                                       |
| Fragment IP data into 16 bytes | `-ff`                                                      |

| Option                    | Purpose                                |
|-----------------------------|-------------------------------------------|
| `--source-port PORT_NUM`    | Specify source port number                |
| `--data-length NUM`         | Append random data to reach the given length |

| Option    | Purpose                                  |
|-----------|--------------------------------------------|
| `--reason`  | explains how Nmap made its conclusion    |
| `-v`        | verbose                                   |
| `-vv`       | very verbose                              |
| `-d`        | debugging                                 |
| `-dd`       | more details for debugging                |

## Nmap Post Port Scans
After discovering ports, we focus on service detection, OS detection, nmap scripting engine and saving scan results.  
| Option                     | Meaning                                            |
|------------------------------|-------------------------------------------------------|
| `-sV`                        | Determine service/version info on open ports          |
| `-sV --version-light`        | Try the most likely probes (2)                        |
| `-sV --version-all`          | Try all available probes (9)                          |
| `-O`                         | Detect OS                                              |
| `--traceroute`                | Run traceroute to the target                          |
| `--script=SCRIPTS`           | Nmap scripts to run                                    |
| `-sC` or `--script=default`  | Run default scripts                                    |
| `-A`                          | Equivalent to `-sV -O -sC --traceroute`                |
| `-oN`                         | Save output in normal format                           |
| `-oG`                         | Save output in a grepable format                       |
| `-oX`                         | Save output in XML format                              |
| `-oA`                         | Save output in normal, XML and Grepable formats        |

