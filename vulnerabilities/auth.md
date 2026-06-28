# Week 3 — Authentication vulnerabilities

## What is authentication vs authorization

Authentication is making sure someone is who they claim to be. Authorization is what that person is allowed to do once they are in. Both matter in AppSec. A lot of vulns happen when apps confuse the two or skip steps in one of them.

---

## Password-based authentication

### Username enumeration

Username enumeration is trying usernames until you find a real one. The app leaks this without realizing it through:

**Different error messages:** "incorrect username" vs "incorrect password" tells you the username exists.

**Response timing:** if one username causes a noticeably longer response, the app is probably running a full password check like bcrypt, which means the username is real. Invalid usernames get rejected early with no hashing at all.

Why attackers care: once you know a valid username you have cut the brute force problem in half.

---

### Brute force protection and how to bypass it

**Account lockout**
The app locks your account after N failed attempts. Bypass: the lockout itself leaks info because if the account gets locked, the username is real.

**IP blocking**
The app blocks your IP after too many failed tries. Bypass: add an X-Forwarded-For header to your request and change the value each time. The app trusts this header and thinks you are coming from a different IP.

**Flawed counter reset**
The app keeps a counter of failed attempts per session or account. Bypass: log in with your own valid credentials between every brute force attempt. A successful login resets the counter so you can keep going indefinitely without ever hitting the limit.

---

## Multi-factor authentication

### What MFA is supposed to do

After entering your password, the app sends a verification code as a second check. The idea is that even if someone has your password they still cannot get in without the second factor.

### 2FA bypass: broken flow enforcement

The flaw: the app authenticates in two steps but does not check that step 2 was actually completed before granting access.

How to exploit it: after entering valid credentials on step 1, just navigate directly to the home page or any authenticated endpoint. The app already considers you logged in and gives you access. The 2FA page was just cosmetic.

Why it happens: the app sets some session state after step 1 that is enough to pass authorization checks even though step 2 was never done.

### 2FA bypass: username in the verification request

The flaw: the verification code request includes the username as a user-controlled parameter. The code is tied to the session but the app does not verify that the username in the request matches the logged-in session.

How to exploit it:
1. Log in with your own valid credentials to get through step 1
2. When the app sends you to the 2FA page intercept the request it makes for your verification code
3. Change the username parameter to the victim's username
4. The app generates a code for the victim's account and you can now access it without knowing their password or their code

Why it happens: the app trusts user input to decide whose account to verify instead of using the server-side session.

---

## Labs completed

Username enumeration via different responses
Username enumeration via subtly different responses
Username enumeration via response timing
Broken brute-force protection via X-Forwarded-For bypass
Username enumeration via account lock
2FA simple bypass via direct navigation
2FA broken logic via username parameter manipulation

---

## Key takeaways

Apps leak more than they think through error messages and timing.
Brute force protections are often bypassable because they trust client controlled inputs like IP headers and counters.
MFA only works if the app actually enforces that every step was completed before granting access.
Never trust user controlled input to determine whose session or account an action applies to.

---

## Other authentication mechanisms

### Stay logged in cookies

A lot of apps implement a remember me feature using a persistent cookie. The cookie is supposed to be impossible to guess but some apps build it from predictable values.

In the lab the cookie was just base64 encoded and decoded to username:password where the password was md5 hashed. Since we could log in with our own account and study our own cookie we figured out the formula. Then we built a payload that takes every password from a wordlist, hashes it with md5, base64 encodes username:hash, and sends it as the cookie value until the app accepts one.

Why it is weak: the construction is predictable, the encoding is reversible, and md5 with no salt means common passwords are crackable instantly.

### Offline password cracking

If an attacker cannot brute force directly, they can still crack the cookie offline if they can get it first.

In the lab we used XSS to steal the victim's stay-logged-in cookie. We decoded it from base64 and got the md5 hash of their password. We pasted the hash into crackstation which has precomputed hashes for millions of common passwords and it returned the original password instantly.

This works because md5 with no salt produces the same hash every time for the same input. Sites like crackstation just store a giant lookup table of password to hash pairs.

### What salt is and why it matters

Salt is a random string the app adds to your password before hashing it. Instead of hashing `password123` the app hashes `rAnD0mStRiNg_password123` and stores the salt alongside the hash.

This kills precomputed lookup tables completely because even if two users have the same password their hashes are different. The offline cracking lab worked specifically because there was no salt so the md5 hash of a common password was already sitting in crackstation's database.

---

## Labs completed (updated)

Username enumeration via different responses
Username enumeration via subtly different responses
Username enumeration via response timing
Broken brute-force protection via X-Forwarded-For bypass
Username enumeration via account lock
2FA simple bypass via direct navigation
2FA broken logic via username parameter manipulation
Brute-forcing a stay-logged-in cookie
Offline password cracking via XSS and hash lookup

---

## Key takeaways (updated)

Apps leak more than they think through error messages and timing.
Brute force protections are often bypassable because they trust client controlled inputs like IP headers and counters.
MFA only works if the app actually enforces that every step was completed before granting access.
Never trust user controlled input to determine whose session or account an action applies to.
Predictable cookie construction is dangerous even if the values are encoded or hashed.
MD5 with no salt is effectively plaintext for common passwords because of precomputed lookup tables.
Salt makes hashes unique per user and kills offline cracking via lookup tables.

---

## Password reset mechanisms

### Sending passwords by email

Some apps email the user a newly generated password. This is bad practice because email is not a secure channel, inboxes persist across devices, and if the password does not expire immediately it is sitting there waiting to be intercepted by a man in the middle attack.

### Resetting via URL

Better apps generate a unique high entropy token and email a reset link containing it. The token should have no hints about which user it belongs to, should expire quickly, and should be destroyed immediately after use.

**Weak implementation: guessable parameter**
Some apps use a predictable parameter like:
```
http://vulnerable-site.com/reset-password?user=carlos
```
An attacker just changes the username and resets anyone's password directly.

**Weak implementation: token not validated on submission**
The app checks the token when you visit the reset page but forgets to check it again when you actually submit the new password. The flaw is that the token is only validated in an early step and not carried through the whole process.

How to exploit it: visit the reset page with your own valid token, intercept the form submission, change the username parameter to the victim, and submit. The app resets the victim's password without ever checking if your token belongs to them.

**Password reset poisoning via middleware**
Apps sometimes build the reset link dynamically using the Host or X-Forwarded-Host header. The backend does something like:

```python
reset_link = "https://" + request.headers.get("X-Forwarded-Host") + "/forgot-password?token=" + token
```

So whatever domain you put in X-Forwarded-Host ends up in the email sent to the victim.

How to exploit it:
1. Send the forgot password request for the victim with your exploit server in the X-Forwarded-Host header
2. The app generates a real valid token for the victim and emails a reset link pointing to your server
3. The victim clicks it and your server logs the GET request including the real token as a query parameter
4. You take that token and use it on the legitimate reset URL to set a new password

The request to your server does nothing functionally. It is just a GET request that your server logs. But that log contains the real token which you then use on the real endpoint.

Important syntax note: the header value is just the bare domain with no https:// prefix because the app prepends that itself:
```
X-Forwarded-Host: your-exploit-server.net
```
Not:
```
X-Forwarded-Host: https://your-exploit-server.net
```

---

### Password change functionality

Password change pages are vulnerable when they let you control the username through a hidden field or request parameter. If the app does not verify that the username matches the logged in session, you can target any user.

**Brute force via password change**
The password change page had an interesting behavior. If you enter the wrong current password with two different values for the new password fields, the app tells you the current password is incorrect. But if you enter the correct current password with two different new password values, it tells you the new passwords do not match. That different response reveals when you have found the real password.

This also bypasses account lockout because the lockout is tied to the login page. The password change page has no such protection.

How to exploit it: brute force the current password field while keeping the two new password fields intentionally different. Watch for the response that changes from incorrect password to passwords do not match. That is the real password.

---

## How to secure authentication

**Credentials and login**
Never send or store plaintext passwords. Always hash with bcrypt, scrypt, or argon2 which are slow by design and make brute forcing expensive. Always use salt so identical passwords produce different hashes.

**Brute force protection**
Rate limit login attempts by both IP and account. Do not trust X-Forwarded-For or X-Forwarded-Host headers unless you control the proxy. Apply the same limits to every authentication surface including password reset and password change, not just the login page.

**Username enumeration**
Return identical error messages, identical status codes, and identical response times for invalid usernames and invalid passwords. Use asynchronous processing to normalize timing so an attacker cannot tell the difference.

**Password reset**
Use high entropy unpredictable tokens. Never use guessable parameters like username in the reset URL. Validate the token at every step of the process not just when the page first loads. Expire tokens quickly and destroy them immediately after use. Never build reset links using user controlled headers like X-Forwarded-Host.

**MFA**
Enforce that every step of the auth flow is completed before granting access. Never use user controlled input to determine whose verification code to generate or validate.

**General**
Audit every authentication related endpoint equally, not just the main login page. Password reset, password change, and account recovery are all part of the attack surface.

---

## Labs completed (final)

Username enumeration via different responses
Username enumeration via subtly different responses
Username enumeration via response timing
Broken brute-force protection via X-Forwarded-For bypass
Username enumeration via account lock
2FA simple bypass via direct navigation
2FA broken logic via username parameter manipulation
Brute-forcing a stay-logged-in cookie
Offline password cracking via XSS and hash lookup
Password reset broken logic
Password reset poisoning via middleware
Password brute-force via password change