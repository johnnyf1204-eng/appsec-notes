# Cross-Site Scripting (XSS)

## What is it
When an app takes user input and puts it into a page without sanitizing it, letting you inject a script that runs in another user's browser.

## Why it works
The browser can't tell the difference between the site's legitimate JavaScript and yours. If your input ends up in the HTML unescaped, the browser just executes it.

## The three types

### Reflected XSS
The payload is in the URL. The attacker sends the victim a crafted link and the server reflects it back in the response.

**The catch:** the attacker is praying the victim is logged in when they click. No guarantee. It's a shot in the dark — send the link and hope.

### Stored XSS
The payload gets saved in the database (comment, profile field, post) and fires every time someone loads that page.

**Why it's worse:** the attacker already knows users are going to see it. They're logged in by definition — they're browsing the site. No praying required. More reliable, more dangerous.

### DOM XSS
The vulnerability lives in client-side JavaScript, not the server. The page's own JS reads from something attacker-controlled (like `location.search`) and writes it into the DOM unsafely.

## My insight
Reflected vs stored isn't just a technical difference — it's a reliability difference from the attacker's perspective. Reflected is a gamble. Stored is a guarantee. That's why stored is treated as higher severity.

## How to find it
- Try `<script>alert(1)</script>` in any input field
- If alert fires, you have XSS
- Check: search bars, comment boxes, profile fields, anything that gets displayed back to users

## Labs completed
- Reflected XSS into HTML context, no encoding — injected `<script>alert(1)</script>` in search, fired immediately
- Stored XSS into HTML context, no encoding — injected payload in comment field, fires on every page load for every visitor

## How to fix it
- Encode output — convert `<` to `&lt;` so the browser treats it as text, not code
- Content Security Policy (CSP) as a second layer
- Never trust user input that gets rendered into the page