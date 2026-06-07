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

- **Source** — where attacker-controlled input comes from (e.g. `location.search`, `location.hash`, `document.cookie`)
- **Sink** — where that input ends up being executed or rendered (e.g. `document.write`, `innerHTML`, `eval`, `$()`)

The vulnerability exists when there's a path from source to sink with no sanitization in between. The sink determines what payload you use ,different sink, different approach, same thinking.

---

## The mental model for every DOM XSS lab

1. Find the source — where is input coming from?
2. Find the sink — where does it end up?
3. Identify the context  what's wrapping your input in the DOM?
4. Close everything wrapping you
5. Inject

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

### jQuery attr() — href attribute

Sets an HTML attribute value. Input is treated as a URL, not raw HTML so breaking out with tags doesn't work.
Use `javascript:` protocol instead , browsers execute JS when a link with this href is clicked.

```
?returnPath=javascript:alert(document.domain)
```

Only works on attributes that accept URLs that navigate/load pages (href, action, iframe src).
Doesn't work on img src  that just loads a resource.

### jQuery $() — selector sink

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
- Problem: `hashchange` only fires on change, not on load  can't just send victim a crafted URL
- Solution: deliver via iframe

**Why the iframe:**

```html
<iframe src="https://vulnerable-site.com#" onload="this.src+='<img src=1 onerror=alert(1)>'">
```

1. Iframe loads vulnerable page with empty hash `#`
2. `onload` fires after page fully loads
3. Payload gets appended to hash — hash changes from `#` to `#<img src=1 onerror=alert(1)>`
4. `hashchange` fires on the vulnerable page
5. `$()` receives the hash, sees HTML, injects it into the DOM
6. `onerror` fires, alert executes

The iframe lives on the attacker's page. Victim just visits the attacker's page  the vulnerable site loads invisibly in the background. Stealthier than reflected XSS because no suspicious URL appears in the victim's address bar.

### AngularJS {{}} expression injection

When a page uses `ng-app`, AngularJS takes control of that element and evaluates anything inside `{{}}` as JavaScript.

- Look for `ng-app` attribute in page source (Ctrl+U)
- If your input lands inside an `ng-app` element, inject `{{2+2}}` to confirm — if page renders `4` you have injection
- AngularJS sandboxes direct JS calls like `alert(1)` need a sandbox escape
- Classic escape using Angular's own internals:

```
{{$on.constructor('alert(1)')()}}
```

- `$on` — an internal Angular function that's accessible
- `.constructor` — every function's constructor is the built-in `Function` object
- `Function('alert(1)')` — builds a new function from a string (same idea as eval)
- `()` — calls it immediately

No angle brackets or event handlers needed you're injecting an expression, not HTML.

---

## Real-world insight: dead code is dangerous

The `document.write` lab had a URL parameter (`storeId`) that was never used in normal site flow  the stock checker used POST instead. But the JS that reads `storeId` from the URL was still there, forgotten. Dead/orphaned code sitting in production, harmless looking, until someone reads the source carefully.

In real bug bounties this is a common find ,developers change approach and forget to clean up old code. The vulnerability isn't in obviously bad code. It's in reasonable features with slightly wrong assumptions.

---

## Why if(store) doesn't protect anything

```javascript
if(store) {
    document.write('<option selected>'+store+'</option>');
}
```

This only checks **does a value exist** , not whether it's valid. A real fix uses whitelist validation:

```javascript
var validStores = ["London","Paris","Milan"];
if(validStores.includes(store)) {
    document.write('<option selected>'+store+'</option>');
}
```

Most injection vulnerabilities exist not because there's zero validation but because the validation checks the wrong thing.

---

## My insights

Reflected vs stored isn't just a technical difference , it's a reliability difference from the attacker's perspective. Reflected is a gamble. Stored is a guarantee. That's why stored is treated as higher severity.

Injection attacks all share the same root cause: the boundary between data and code broke down. The developer's mistake is always the same 
,trusting user data and mixing it with code without checking it first. SQLi, XSS, command injection — same concept, different context.

---

## How to find it

- Try `<script>alert(1)</script>` in any input field
- If alert fires, you have XSS
- In DOM XSS: search source for `document.write`, `innerHTML`, `eval`, `$()` /trace where input flows
- Use Ctrl+F in Sources tab to find sinks
- Add test input to URL, search for it in Elements tab to find where it lands in the DOM

---

## How to fix it

- Encode output — convert `<` to `&lt;` so browser treats it as text not code
- Whitelist validation — check input is exactly what you expect, not just that it exists
- Use safe sinks — `textContent` and `innerText` treat input as plain text, never parse as HTML
- Content Security Policy (CSP) as a second layer
- Never trust user input that gets rendered into the page

---

## Labs completed

### Reflected & Stored XSS
- Reflected XSS into HTML context (no encoding)
- Stored XSS into HTML context (no encoding)

### DOM XSS (core sinks)
- DOM XSS in `document.write` using `location.search`
- DOM XSS in `innerHTML` using `location.search`
- DOM XSS in `document.write` inside a `<select>` element using `location.search`

### jQuery-based DOM XSS
- DOM XSS in jQuery anchor `href` attribute sink using `location.search`
- DOM XSS in jQuery selector sink using hashchange event

### Framework-based XSS
- DOM XSS in AngularJS expression with encoded characters