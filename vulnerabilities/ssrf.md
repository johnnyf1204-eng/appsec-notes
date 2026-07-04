# Server-Side Request Forgery (SSRF)

## Overview

SSRF is a vulnerability that allows an attacker to induce the server to make HTTP requests to unintended locations. Rather than the attacker making the request directly, the server does it — which means the request carries the server's trust level, bypassing access controls that would normally block external users.

Common real-world triggers: stock checkers, webhooks, URL preview features, analytics that fetch the `Referer` header, and any parameter that accepts a URL.

---

## Why It's Dangerous

Internal systems trust requests coming from the application server. An attacker exploiting SSRF doesn't hit those systems directly — they make the server do it. This means:

- Access control layers sitting in front of the app get bypassed entirely
- Internal APIs with no authentication become reachable
- Admin interfaces on non-public ports become accessible
- The entire internal network becomes a potential attack surface

---

## Attack Vectors

### Loopback / Against the Server Itself

The simplest form — redirect the vulnerable parameter to `localhost` or `127.0.0.1`. Since the request originates from the machine itself, internal-only endpoints open up.

```
stockApi=http://localhost/admin
```

### Internal Network Scanning

The server can reach private IP ranges that external users cannot. By iterating over an internal subnet, it's possible to map live hosts and exposed services.

```
stockApi=http://192.168.0.§1§/admin
```

Fuzz the last octet and look for response length differences — those indicate live hosts.

---

## Bypassing Defenses

### Blacklist Bypass

Applications often block strings like `127.0.0.1`, `localhost`, or `/admin`. Common bypasses:

| Technique | Example |
|-----------|---------|
| Decimal IP | `http://2130706433/` |
| Octal IP | `http://017700000001/` |
| Short form | `http://127.1/` |
| URL encoding | `http://127.0.0.1%2Fadmin` |
| Case variation | `http://LOCALHOST/` |
| Open redirect chain | `http://allowed.com/redirect?url=http://127.0.0.1/admin` |

### Whitelist Bypass

Some applications only allow URLs that contain a specific trusted domain. The bypass surface here is URL parsing inconsistency — exploiting the difference between how the validator reads a URL vs how the HTTP client reads it.

**Embedded credentials (`@`)**

In the URL spec, everything before `@` is credentials, everything after is the host. Some validators get confused about which part to check.

```
http://localhost@stock.weliketoshop.net/
```

**Fragment identifier (`#`)**

Everything after `#` is a fragment — client-side only, never sent to the server. Parsers stop reading the host at `#`.

```
http://localhost#stock.weliketoshop.net/
```

**DNS subdomain trick**

If the whitelist checks whether the trusted domain appears anywhere in the URL string, register a domain that contains it as a subdomain.

```
http://stock.weliketoshop.net.attacker.com/
```

**Open redirect chaining**

If a whitelisted domain contains an open redirect, use it to bounce to the internal target. The validator sees the allowed domain; the server follows the redirect.

```
stockApi=http://stock.weliketoshop.net/redirect?url=http://192.168.0.68/admin
```

---

## Parser Differential — Double URL Encoding

The most sophisticated whitelist bypass technique. The attack exploits the fact that two different components in the request pipeline decode the URL at different stages, and therefore disagree on what it says.

### How it works

`#` URL-encoded is `%23`. Double-encoding the `%` gives `%2523`.

```
http://localhost%2523@stock.weliketoshop.net/admin
```

**Whitelist layer** decodes once:
- `%2523` → `%23` (literal characters, not a special symbol)
- Sees `localhost%23` as credentials, `stock.weliketoshop.net` as the host
- Whitelist passes ✓

**HTTP client** decodes again:
- `%23` → `#` (actual fragment delimiter)
- `#` terminates the host — reads `localhost` as the host, discards everything after
- Connects to `localhost` ✓

Same URL. Two decodings. Two different hosts. The gap between them is the attack.

### Why This Matters Beyond SSRF

Parser differentials are a class of vulnerability, not just an SSRF trick. The same underlying concept drives:

- **HTTP request smuggling** — frontend and backend disagree on where one request ends
- **XSS filter bypasses** — sanitizer and browser parse HTML differently
- **Path traversal bypasses** — validator normalizes the path differently than the filesystem

Any time a security control and an executor process the same input through different parsers, a differential may exist.

---

## Blind SSRF

### What Makes It Blind

In regular SSRF, the server fetches the URL and the response is visible — data from internal systems comes back in the HTTP response. In blind SSRF, the server still makes the request, but the response is never returned to the attacker. The behavior is one-way.

### Detection — Out-of-Band (OAST)

Since there's no response to read, detection relies on monitoring for outbound connections. The approach: point the server at a host you control and watch for incoming requests.

Tools:
- **Burp Collaborator** — generates unique subdomains, logs all DNS and HTTP interactions (Pro only)
- **interactsh** — open source alternative, same concept

If the server makes a request to the callback domain, SSRF is confirmed regardless of what the application returns.

**Common injection point for blind SSRF:** the `Referer` header. Analytics software often fetches URLs in `Referer` to track referring sites.

### DNS Hit, No HTTP

It's common to see a DNS lookup reach Collaborator but no HTTP request follow. This means the application resolved the domain (DNS is rarely blocked) but network-level filtering blocked the outbound HTTP connection. This still confirms the vulnerability exists — it just means exploitation is constrained.

### Exploitation Paths

Blind SSRF can't be used to directly read internal data. However:

- **Internal port scanning** — different DNS/HTTP response behavior reveals live hosts and open ports
- **Payload delivery** — fire known exploit payloads (e.g. Shellshock) at internal services via headers
- **RCE via malicious responses** — if the server's HTTP client can be made to process an attacker-controlled response, client-side vulnerabilities in the HTTP implementation may be exploitable

---

## Attack Surface — Where to Look

| Location | Example |
|----------|---------|
| URL parameters | `stockApi=`, `url=`, `webhook=`, `callback=`, `next=` |
| `Referer` header | Analytics software fetching referring URLs |
| XML/JSON body | URLs embedded in data formats (can chain with XXE) |
| Partial URLs | Hostname-only params assembled server-side |
| `Host` header | Less common, context-dependent |

---

## Labs

| Lab | Difficulty | Technique |
|-----|-----------|-----------|
| Basic SSRF against the local server | Apprentice | `localhost/admin` via stockApi |
| Basic SSRF against another back-end system | Apprentice | Internal IP range scan |
| SSRF with blacklist-based input filter | Practitioner | `127.1` to bypass localhost/127.0.0.1 block |
| SSRF with filter bypass via open redirection | Practitioner | Open redirect on whitelisted domain |
| SSRF with whitelist-based input filter | Expert | Double URL encoding — parser differential |
| Blind SSRF with out-of-band detection | Practitioner | OAST via Referer header (Collaborator required) |

---
