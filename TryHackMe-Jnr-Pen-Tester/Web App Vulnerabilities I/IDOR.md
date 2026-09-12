## IDOR
when an application allows a user to supply a reference for an object but doesn't check whether the user is permitted to access it.  

## IDORs in Encoded IDs
Some identifiers are encoded before being included in query strings, POST data, or cookies. Attackers can decode first, then encode their payload.  

## IDORs in Hashed IDs
Some applications hash identifiers before including them in requests. However, if sequential integers are used. This can be easily hacked by attackers. 

## IDORs in Unpredictable IDs
Some applications will use randomly generated identifiers that cannot be guessed or enumerated. But even with unpredictable IDs, if user is still able to request access to referenced objects without verification IDOR risks still exist.  
Two-Account Technique:
1. Create two accounts (A and B)
2. Log into A and record IDs associated with resources (objects).
3. Log into B and request objects using ID from A.
If server returns A's data, the endpoint lacks authorisation and verification. IDOR risks exist.

## Where are IDORs located
- Background Requests  
  Findable through **Network** tab.
- Javascript files  
  Reviewing these files can reveal endpoints that can be interacted with
- Parameter mining  
  Some endpoints accept parameters that the front end never sends.
- Common Locations  
  query string parameters, POST body data, cookie values, HTTP request headers, REST API path segments (such as /api/users/123/orders), and background AJAX requests. 
