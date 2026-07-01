# CSRF: Cross-Site Request Forgery

## What is CSRF?

CSRF is when an attacker tricks a victim's browser into performing an action on a site where the victim is already authenticated, without the victim knowing. The server can't tell the difference between a legitimate request and a forged one because both come from the same authenticated browser.

## The 3 conditions for CSRF to work

All three need to be true at the same time:

1. A relevant action exists: something worth abusing like changing email, transferring funds, changing password
2. Cookie based session handling: the app uses cookies to identify the user, so the browser sends them automatically with every request
3. No unpredictable request parameters: the request doesn't contain anything the attacker can't know or forge in advance

If any one of these is missing, CSRF doesn't work.

## CSRF vs XSS

| | XSS | CSRF |
|---|---|---|
| What it does | Makes the victim's browser execute malicious JavaScript | Makes the victim's browser perform an action that benefits the attacker |
| Requires JS? | Yes | No |
| Runs in victim's browser? | Yes | Yes, but just a request, no script execution |

XSS is about injecting and running code. CSRF is about forging a legitimate looking request.

---

## Token bypass: validation only happens if token is present

The server validates the token but only if you send one. If you remove the parameter entirely, the server skips validation and accepts the request.

Bypass: drop the csrf parameter from the request completely.

Lab: solved. Removed the csrf parameter from the change email request entirely and the server processed it anyway.

## Token bypass: token not tied to user session

The server checks that the token is valid, meaning it generated it at some point, but doesn't check that it was issued to this specific session. That means you can grab your own valid CSRF token and use it in an attack against someone else.

Bypass: log in as yourself, grab your CSRF token, and use it in your exploit targeting the victim.

Lab: solved. Logged in as myself, captured my own valid token, and reused it inside the exploit aimed at the victim's session.

## Token bypass: validation depends on request method

The app only validates CSRF tokens on POST requests. GET requests go through unchecked.

Bypass: change the request from POST to GET. If the server processes it anyway, the CSRF token check never runs.

Lab: solved. Switched the form method from POST to GET and the token check was skipped entirely.

## Basic CSRF: no defenses at all

No CSRF protection at all. Built a simple HTML form that auto submits a POST request to change the victim's email. The browser sends the victim's session cookie automatically and the server accepts it.

```html
<form action="https://target.com/email/change" method="POST">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit()</script>
```

No token needed. No user interaction needed beyond loading the page.

Lab: solved.

## CSRF cookie injection

Some apps store the CSRF token in a cookie instead of tying it server side to the session. If you can inject your own matching cookie into the victim's browser, you control both halves of the validation pair.

Two step attack:

1. Inject the csrf cookie into the victim's browser via a crafted URL that triggers a Set-Cookie header
2. Submit the forged form with a token that matches the injected cookie

The interesting part is that you control both the cookie and the token so you can make them match, but the server thinks it's a legitimate pair. It shows that CSRF protection is only as strong as the binding between the token and the session.

Lab: solved, both variations of this attack.

## Key takeaway on tokens

CSRF tokens only work if the server validates them and ties them to a specific session. Any gap in that logic, whether missing validation, wrong method, or weak binding, and the protection falls apart.

Real defense: use a token that is cryptographically bound to the session, validate it on every state changing request regardless of method, and consider SameSite cookies as a second layer.

---

## SameSite cookies: site vs origin

A site is the eTLD+1 (effective top level domain plus one extra level), so app.example.com and intranet.example.com are the same site. An origin needs the exact same scheme, domain, and port. This matters because a request can be cross origin but still same-site, so SameSite cookies will still be sent even between different subdomains. Any XSS on one subdomain can be abused to attack other subdomains on the same site, since they all share the same-site relationship.

## SameSite cookies: Strict vs Lax vs None

Strict: cookie is never sent on cross-site requests, full stop.

Lax: cookie is sent on cross-site requests only if it's a GET request AND it's a top-level navigation (like clicking a link). Cross-site POSTs, iframes, and background script requests don't get the cookie. This is Chrome's default if no SameSite attribute is set.

None: cookie is sent on every request regardless of origin. Must be paired with the Secure attribute or the browser rejects it.

## SameSite bypass: GET method override

Some servers don't actually care whether a request comes in as GET or POST, they just read a hidden form field like `_method=POST` and treat it as a POST internally for routing. Since the cookie is Lax and the actual HTTP request is a GET top-level navigation, the SameSite restriction lets the cookie through, even though the server ends up processing it as if it were a POST.

Lab: solved. Built a form with method GET but added a hidden `_method=POST` field so the Lax cookie was sent while the server processed it as a POST.

## SameSite bypass: client side redirect (Strict cookies)

A server side redirect is recognized by the browser as originating from the original cross-site request, so the Strict restriction still applies on the follow-up. A client side redirect (like JS reading a URL parameter and setting location.href) is invisible to the browser's SameSite logic, the resulting request just looks like a normal same-site request fired from a page you're already on, so the Strict cookie gets attached.

Lab: solved. Found a redirect gadget on the site that used a URL parameter to decide where to send the user next, pointed it at the sensitive endpoint, which made the browser treat it as a same-site request and attach the Strict cookie.

## SameSite bypass: sibling domain

Same-site doesn't mean same-origin. If one subdomain (sibling) has any vulnerability that lets you trigger a request, like XSS, you can use it to fire requests against another subdomain on the same site, and since SameSite restrictions only check the site not the specific origin, the cookie still gets sent. So securing your main domain isn't enough if a sibling subdomain is weak.

Lab: not solved yet. Need to revisit, the idea is finding a flaw on a sibling subdomain to fire the actual attack request from there instead of from an external attacker site.

## SameSite bypass: cookie refresh during the grace period

When a user logs in, some browsers temporarily relax SameSite restrictions for that fresh cookie for about 2 minutes (this only applies to cookies that get the Lax default automatically, not ones explicitly set with SameSite=Lax). During that short window, cross site requests that would normally get blocked are allowed through. So if you can force the victim to relogin and then fire your CSRF request immediately after, you land inside that window.

Exploit logic, step by step:

1. Force the victim to visit the login page so their session cookie refreshes. This can't happen automatically on page load because browsers block popups unless they're triggered by a real user interaction.
2. So instead, the popup only opens when the victim clicks anywhere on the page. This satisfies the browser's popup blocker requirement.
3. Once the popup opens and the OAuth flow runs in the background, the victim's session cookie gets refreshed.
4. At the same time, a timer is running (setTimeout) so the actual CSRF form submission waits a few seconds before firing, giving the OAuth flow enough time to finish and set the new cookie.
5. When the timer runs out, the form submits. Since it's happening within the grace period of the fresh login, the browser still attaches the session cookie even though it's a cross site POST.

The bug in my first attempt: the form submission code sat outside the click handler entirely, just placed after it in the script. That meant it ran 5 seconds after the page loaded, completely disconnected from whether the victim had clicked or not, so the popup and the OAuth refresh never had a chance to happen before the form fired.

The fix: move the setTimeout call inside the onclick handler itself, so the timer only starts counting once the victim actually clicks, which is also the same moment the popup opens and the relogin begins. That way the 5 second wait actually overlaps with the OAuth flow finishing.

Working exploit:

```html
<script>
window.onclick = () => {
    window.open('https://YOUR-LAB-ID.web-security-academy.net/social-login');
    setTimeout(f1, 5000);
}
function f1() {
    document.forms[0].submit();
}
</script>
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="anything@web-security-academy.net">
</form>
```

Lab: solved.

---

## Referer-based defenses

Some apps skip CSRF tokens entirely and instead validate the HTTP Referer header, checking that the request came from their own domain. This is weaker than token-based protection and has two common bypass patterns.

## Referer bypass: validation only happens if header is present

The app validates the Referer when it exists but skips validation entirely if the header is missing. So just strip it.

Bypass: add a meta tag to your exploit page that tells the browser not to send the Referer header at all.

```html
<meta name="referrer" content="never">
```

The browser drops the header, the server sees no Referer, skips the check, and processes the request.

Lab: solved.

## Referer bypass: naive string matching

The app checks that the Referer contains or starts with the target domain, but doesn't parse it properly. That means you can smuggle the expected domain into a part of the URL the app isn't actually checking as a domain.

Two variants:

If the app checks that Referer starts with the expected domain, put the target domain as a subdomain of your attacker domain:

```
http://vulnerable-website.com.attacker.com/csrf-attack
```

If the app just checks that the Referer contains the target domain anywhere, put it in the query string:

```
http://attacker.com/csrf-attack?vulnerable-website.com
```

One catch: modern browsers strip the query string from the Referer by default. Fix that by setting this header on your exploit server's response:

```
Referrer-Policy: unsafe-url
```

That forces the browser to send the full URL including query string.

Lab: solved.

---

## Preventing CSRF

**CSRF tokens** are the strongest defense. A valid token needs to be unpredictable, tied to the user's session, and validated on every state-changing request regardless of method. Transmit it as a hidden form field in a POST, never in a cookie, never in the URL. If the token is missing entirely, reject the request the same way you'd reject a bad token.

**SameSite=Strict cookies** as a second layer. Strict is the safest default. Only drop to Lax if you have a real reason. Never use None unless you fully understand what you're opening up.

**Cross-origin same-site attacks** aren't stopped by SameSite at all. Isolate sensitive functionality from anything that handles user-uploaded content or has weaker security, even if it lives on a sibling subdomain.

Takeaway: token validation is the core defense, SameSite is the backup. Both need to be in place because each covers gaps the other leaves open.