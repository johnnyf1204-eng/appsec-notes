
Csrf notes · MD
# CSRF: Cross-Site Request Forgery
 
## What is CSRF?
 
CSRF is when an attacker tricks a victim's browser into performing an action on a site where the victim is already authenticated, without the victim knowing. The server can't tell the difference between a legitimate request and a forged one because both come from the same authenticated browser.
 
## The 3 conditions for CSRF to work
 
All three need to be true at the same time:
 
1. **A relevant action exists**: something worth abusing like changing email, transferring funds, changing password
2. **Cookie-based session handling**: the app uses cookies to identify the user, so the browser sends them automatically with every request
3. **No unpredictable request parameters**: the request doesn't contain anything the attacker can't know or forge in advance
If any one of these is missing, CSRF doesn't work.
 
## CSRF vs XSS
 
| | XSS | CSRF |
|---|---|---|
| What it does | Makes the victim's browser execute malicious JavaScript | Makes the victim's browser perform an action that benefits the attacker |
| Requires JS? | Yes | No |
| Runs in victim's browser? | Yes | Yes, but just a request, no script execution |
 
XSS is about injecting and running code. CSRF is about forging a legitimate looking request.
 
## CSRF token bypasses
 
### 1. Token validation only happens if the token is present
 
The server validates the token but only if you send one. If you remove the parameter entirely, the server skips validation and accepts the request.
 
Bypass: drop the csrf parameter from the request completely.
 
### 2. Token not tied to the user session
 
The server checks that the token is valid meaning it generated it at some point, but doesn't check that it was issued to this specific session. That means you can grab your own valid CSRF token and use it in an attack against someone else.
 
Bypass: log in as yourself, grab your CSRF token, and use it in your exploit targeting the victim.
 
### 3. Token validation depends on the request method
 
The app only validates CSRF tokens on POST requests. GET requests go through unchecked.
 
Bypass: change the request from POST to GET. If the server processes it anyway, the CSRF token check never runs.
 
## Labs completed
 
### Lab 1: basic CSRF, no defenses
 
No CSRF protection at all. Built a simple HTML form that auto submits a POST request to change the victim's email. The browser sends the victim's session cookie automatically and the server accepts it.
 
```html
<form action="https://target.com/email/change" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit()</script>
```
 
No token needed. No user interaction needed beyond loading the page.
 
### Labs 2 to 4: token bypass variations
 
Each lab had the same goal (change victim's email) but with a different flawed token implementation:
 
- One validated token only if present, so the parameter got dropped
- One issued tokens not bound to sessions, so own token was used against victim
- One only checked POST, so switching to GET bypassed it
### Labs 4 and 5: CSRF cookie injection
 
These required injecting a CSRF cookie into the victim's browser first, then using a matching token in the forged request. Two step attack:
 
1. Inject the csrf cookie into the victim's browser via a crafted URL that triggers a Set-Cookie header
2. Submit the forged form with a token that matches the injected cookie
The interesting part is that you control both the cookie and the token so you can make them match, but the server thinks it's a legitimate pair. It shows that CSRF protection is only as strong as the binding between the token and the session.
 
## Key takeaway
 
CSRF tokens only work if the server validates them and ties them to a specific session. Any gap in that logic, whether missing validation, wrong method, or weak binding, and the protection falls apart.
 
Real defense: use a token that is cryptographically bound to the session, validate it on every state changing request regardless of method, and consider SameSite cookies as a second layer.