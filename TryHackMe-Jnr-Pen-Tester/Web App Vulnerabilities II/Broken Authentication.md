## Types of Authentication Bypass
### Username Enumeration
used to produce a list of accounts registered on the target application.  
Involves submitting potential usernames to a form that responds differently for registered and unregistered values.  
e.g. this username is already used  
### Credential Brute Force
uses a list together with a dictionary of common passwords to find accounts whose passwords can be guessed.  
### Logic Flaws
Occur when the intended flow of an authentication-related workflow can be redirected by valid-looking input. Password reset and account recovery workflows are a frequent source of such flaws, because they commonly split input across multiple HTTP parameters, cookies, and session variables. A mistake in how those inputs are combined can allow the reset link for one account to be delivered to a different address than the one on file.
### Cookie Manipulation
HTTP is a stateless protocol. Therefore, cookies are used to ensure authenticated users do not have to supply credentials on every request. This can be manipulated, by modifying cookies we can gain unauthorised access.  
Cookie Formats:
- Plaintext cookies
  -   can be easily manipulated
- Hashed Cookies
  - Simple hashes can be reversed
- Encoded Cookies
  - Decoding cookie
  - Changing values
  - Encoding cookie
