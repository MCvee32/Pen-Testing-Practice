## SSRF Examples
### Full URL in Parameter  
- application takes value of server parameter and contrasts a request
- an attacker can replace this value to redirect the request
`server=server.website.thm/flag?id=9&x=`
- &x= neutralises what the application appends to the URL (treats as string).
**Resulting Server-Side Request**: https://server.website.thm/flag?id=9&x=/api/item?id=2

### Partial URL (Hostname or Path Only)
Applications that only accept a hostname or path segment and construct the rest of the URL server side  
`https://website.thm/stock?server=api.internal`  
Can be replaced with:  
`https://website.thm/stock?server=attacker.com`  

### Path Traversal
an attacker controls only a path segment, directory traversal can be used.  
`https://website.thm/stock?url=/item/123/details`
Attacker can supply `/../admin`  
**Request**: https://website.thm/admin

### Hidden Form Fields
These are visible from the page's HTML source and only discoverable through manual inspection.  
example avatar feature:  
`<input type="hidden" name="avatar" value="/images/avatars/default.png">`  
the value filed can be modified to point to a different internal source  
