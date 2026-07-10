# Access Control Vulnerabilities

## Overview

Access control is what determines whether an authenticated user is actually allowed to do what they're trying to do. It sits on top of authentication (proving who you are) and session management (tracking who you are across requests). When it breaks, users can access resources or perform actions beyond their permission level.

Broken access controls are one of the most common critical vulnerabilities in real applications — not because they're technically complex, but because they require humans to make correct design decisions consistently across every endpoint.

Common real-world triggers: parameter-based role checks, direct object references in URLs, multi-step processes where only some steps are protected, and trust placed in client-controlled headers.

---

## Types of Access Control

**Vertical** — restricts functionality by user type. An admin can delete accounts; a regular user cannot. Breaking this means gaining access to functionality above your privilege level.

**Horizontal** — restricts resources by ownership. A user can see their own bank account, not someone else's. Breaking this means accessing another user's data at the same privilege level.

**Context-dependent** — restricts actions based on application state. A user can't modify a shopping cart after payment has been made. Breaking this means performing actions out of the intended sequence.

---

## Vertical Privilege Escalation

### Unprotected Functionality

The simplest case — sensitive endpoints exist but have no access control enforced. The only "protection" is that the link isn't shown to regular users. Anyone who knows or guesses the URL gets in.

```
https://example.com/admin
https://example.com/administrator-panel-yb556
```

Common discovery methods:
- `robots.txt` — developers often list admin paths here to block crawlers, inadvertently revealing them
- JavaScript source — role-based UI logic often embeds admin URLs in client-side code, visible to everyone regardless of role
- Wordlist brute-forcing — tools like `ffuf` or `dirbuster` against common admin path patterns

### Parameter-Based Role Control

The application stores the user's role in a location the user controls — a cookie, hidden field, or query parameter — and makes access decisions based on that value.

```
GET /home?admin=true
GET /home?role=1
```

Since the parameter is user-controlled, it can be modified. The server trusts it.

### Platform Misconfiguration — Header Overrides

Some frameworks support non-standard headers that override the request URL at the platform layer. If front-end access controls check the URL but the backend honors these headers, the restriction is bypassed.

```
POST / HTTP/1.1
X-Original-URL: /admin/deleteUser
X-Rewrite-URL: /admin/deleteUser
```

The front-end sees a request to `/`, which is allowed. The application processes a request to `/admin/deleteUser`.

### Platform Misconfiguration — HTTP Method Override

Access controls sometimes restrict a specific method on a URL (e.g. `POST /admin/deleteUser` is denied for regular users). If the application also accepts the same action via `GET` or another method, the restriction is bypassed by switching methods.

### URL-Matching Discrepancies

The access control layer and the application router may not agree on what constitutes the same endpoint:

| Variant | Example |
|---------|---------|
| Case inconsistency | `/ADMIN/deleteUser` vs `/admin/deleteUser` |
| Trailing slash | `/admin/deleteUser/` vs `/admin/deleteUser` |
| File extension (Spring) | `/admin/deleteUser.anything` matches `/admin/deleteUser` |

If the access control check treats these as different endpoints but the router maps them to the same handler, the check is bypassed.

---

## Horizontal Privilege Escalation

Accessing another user's resources by manipulating a reference to an object the current user doesn't own.

```
GET /myaccount?id=123   → attacker's account
GET /myaccount?id=124   → victim's account
```

### IDOR — Insecure Direct Object References

A subcategory of horizontal access control failures. The application uses a user-supplied value (ID, filename, key) to directly access an object, with no verification that the requesting user owns it.

**Predictable IDs** — sequential integers are trivially enumerable. Increment the ID and access other users' records.

**Unpredictable IDs (GUIDs)** — harder to guess, but GUIDs often leak elsewhere in the application: user messages, reviews, shared content, API responses. Find the GUID in one place, use it in another.

**Data leakage in redirects** — the application detects unauthorized access and issues a redirect to the login page, but the redirect response itself contains the sensitive data in the body before the browser follows it. The redirect happens; the data is still there.

### Horizontal to Vertical Escalation

Horizontal escalation becomes vertical if the targeted object belongs to a privileged user. Accessing an admin's account page via IDOR may expose their password, a password reset mechanism, or direct access to admin functionality — turning a horizontal move into full vertical privilege escalation.

---

## Other Broken Access Control Patterns

### Multi-Step Process Vulnerabilities

Applications that implement sensitive actions across multiple steps often only enforce access control on some of them. If step 3 of a 3-step process isn't protected, an attacker can skip steps 1 and 2 and submit the final request directly.

The assumption that a user "must have passed the earlier steps" to reach a later one is not a security control.

### Referer-Based Access Control

The application enforces access control on `/admin` properly, but for sub-pages like `/admin/deleteUser` it only checks that the `Referer` header contains `/admin`. Since `Referer` is fully attacker-controlled, this is trivially bypassed by forging the header.

```
GET /admin/deleteUser?username=carlos HTTP/1.1
Referer: https://example.com/admin
```

### Location-Based Access Control

Geographic restrictions enforced client-side or via IP geolocation. Bypassed with VPNs, proxies, or manipulation of client-side location data.

---

## Prevention

- **Deny by default** — if access isn't explicitly granted, it's denied
- **Single enforcement mechanism** — access control logic in one place, applied consistently, not duplicated per-endpoint
- **Server-side only** — never trust client-supplied role, permission, or identity values
- **Never rely on obscurity** — hidden URLs, unpredictable IDs, and client-side UI suppression are not access controls
- **Audit multi-step flows** — verify every step in a process enforces access control independently
- **Avoid Referer-based decisions** — the header is attacker-controlled

---

## Labs

| Lab | Difficulty | Technique |
|-----|-----------|-----------|
| Unprotected admin functionality | Apprentice | Admin URL in `robots.txt` |
| Unprotected admin functionality with unpredictable URL | Apprentice | Admin URL leaked in JavaScript source |
| User role controlled by request parameter | Apprentice | `?admin=true` parameter tampering |
| User role can be modified in user profile | Apprentice | Role field editable via profile API |
| User ID controlled by request parameter | Apprentice | IDOR — predictable numeric ID |
| User ID controlled by request parameter, with unpredictable user IDs | Apprentice | GUID leaked in another user's content |
| User ID controlled by request parameter with data leakage in redirect | Apprentice | Sensitive data in redirect response body |
| User ID controlled by request parameter with password disclosure | Apprentice | Horizontal → vertical via admin account IDOR |
| Insecure direct object references | Apprentice | IDOR on static file (chat transcript) |
| URL-based access control can be circumvented | Practitioner | `X-Original-URL` header override |
| Method-based access control can be circumvented | Practitioner | Switch POST to GET on restricted endpoint |
| Multi-step process with no access control on one step | Practitioner | Skip to unprotected final step directly |
| Referer-based access control | Practitioner | Forge `Referer` header to bypass sub-page check |

