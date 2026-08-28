## OS Enumeration
`uname -a` - prints system info (kernel used).  
`cat /proc/version` - gives kernel version or compiler used.  
`cat /etc/issue` - OS info.  
`ps` - running processes.  
`ps` options:  
- aux: processes for all users, display user that launched the process, show processes not attached to a terminal
- axjf: process from all users, processes not attached to a terminal, jobs format output, tree view.
`cron` service
examining `/etc/cron`, `/var/spool/cron/` and `/etc/cron.d/` we can check what's scheduled and who's running it.
`dpkg -l` - lists installed softwares.

## User Enumeration
`id` - user's privilege level and groups.  
`env` - environment variables.  
`history` - commands ran.  
`sudo -l` - all commands user can run using sudo.  
`/etc/passd/` - good way of finding users.  

## Network Enumeration
`netstat` 
`-a` all listening ports and established connections.  
`at` or `au` list TCP or UDP protocols.  
`-l` ports in "listening" mode.  
`-s` network usage stats.  
`tp` connections with the service name and PID.  
`-n` shows numeric addresses and port numbers instead of resolving them.  
`ano` all sockets, do not resolve names, display timers.  

## File Enumeration
`ls -la` list all files and directories, including hidden ones.  
`find`  
### Common `find` Command Examples

- `find . -name flag1.txt` — finds the file named "flag1.txt" in the current directory and subdirectories recursively
- `find /home -name flag1.txt` — finds the file named "flag1.txt" in the `/home` directory and subdirectories recursively
- `find / -type d -name config` — finds the directory named "config" under `/`
- `find / -type f -perm 0777` — finds files with 777 permissions (files readable, writable, and executable by all users)
- `find / -perm -a=x` — finds executable files
- `find /home -user frank` — finds all files for user "frank" under `/home`
- `find / -mtime -10` — finds files that were modified in the last 10 days
- `find / -atime -10` — finds files that were accessed in the last 10 days
- `find / -cmin -60` — finds files changed within the last hour (60 minutes)
- `find / -amin -60` — finds files accessed within the last hour (60 minutes)
- `find / -size +50M` — finds files with at least 50 MB size

### Folders and files that can be written to or executed from:
- `find / -writable -type d 2>/dev/null` finds world-writeable folders.
- `find / -perm -222 -type d 2>/dev/null` finds world-writeable folders.
- `find / -perm -o w -type d 2>/dev/null` finds world-writeable folders.
- `find / -perm -o x -type d 2>/dev/null` finds world-executable folders.

`find / -name pass*.txt` don't know each name of a file.  


