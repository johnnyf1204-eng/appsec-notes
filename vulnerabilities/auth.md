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