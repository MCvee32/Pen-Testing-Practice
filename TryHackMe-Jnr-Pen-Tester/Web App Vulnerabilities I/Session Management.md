## Session Management Lifecycle
**Session Creation:** session value creation, usage and storage  
**Session Tracking:** Session values are submitted with each new request, allowing action tracking  
**Session Expiry:** Session value should have an expiry/lifetime. Submitting old session values should be denied by applications and redirected to login page.  
**Session Termination:** On logout, the application should terminate the user's session  

## Cookies vs Tokens
### Cookie-based Session Management
When the web app wants to begin tracking. In a response, the `Set-Cookie` header value will be sent.  
Other header attributes:
- **Secure**
- **HTTPOnly**
- **Expire**
- **SameSite**

### Token-based Session Management
Relies on client-side code.  
After authentication, the web app provides a token within the request body.  
HTTP header for JWT:  
`Authorisation: Bearer`  

## Securing the Session Lifecycle
### Session Creation
**Weak Session Values**  
**Controllable Session Values**  
e.g. verifying token signatures  
**Session Fixation**  
Rotating session value  
**Insecure Session Transmission**  
e.g. insecure redirect: attacker controlling url redirect after authentication  

### Session Tracking
**Authroisation Bypass**  
Insufficient checks for user permissions  
Vertical Bypass and Horizontal Bypass  
**Insufficient Logging**  

### Session Expiry
Ensuring suitable expiry times  

### Session Termination
Not properly terminating session server-side.
