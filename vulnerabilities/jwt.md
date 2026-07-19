# JWT Attacks

## Overview

JWT (JSON Web Token) attacks exploit design issues and flawed handling of signed tokens used for authentication, session management, and access control. Because JWTs are self-contained — the server doesn't store anything about tokens it issues — a server that fails to properly verify a token's signature has no way to detect tampering. Get this wrong, and an attacker can forge tokens with arbitrary claims: impersonate other users, escalate privileges, or bypass authentication entirely.

The core danger: if you can get the server to accept a token you signed yourself, you own the session layer.

---

## JWT Format

A JWT has 3 parts, separated by dots: **header.payload.signature**

```
eyJraWQiOiI5MTM2ZGRiMy1jYjBhLTRhMTktYTA3ZS1lYWRmNWE0NGM4YjUiLCJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJwb3J0c3dpZ2dlciIsImV4cCI6MTY0ODAzNzE2NCwibmFtZSI6IkNhcmxvcyBNb250b3lhIiwic3ViIjoiY2FybG9zIiwicm9sZSI6ImJsb2dfYXV0aG9yIiwiZW1haWwiOiJjYXJsb3NAY2FybG9zLW1vbnRveWEubmV0IiwiaWF0IjoxNTE2MjM5MDIyfQ.SYZBPIBg2CRjXAJ8vCER0LA_ENjII1JakvNQoP-Hw6GG1zfl4JyngsZReIfqRvIAEi5L4HV0q7_9qGhQZvy9ZdxEJbwTxRs_6Lb-fZTDpW6lKYNdMyjw45_alSCZ1fypsMWz_2mTpQzil0lOtps5Ei_z7mM7M8gCwe_AGpI53JxduQOaB5HkT5gVrv9cKu9CsW5MS6ZbqYXpGyOG5ehoxqm8DL5tFYaW3lB50ELxi0KsuTKEbD0t5BCl0aCR2MBJWAbN-xeLwEenaqBiwPVvKixYleeDQiBEIylFdNNIMviKRgXiYuAvMziVPbwSgkZVHeEdF5MQP1Oe2Spac-6IfA
```

- **Header** — base64url-encoded JSON, metadata about the token (e.g. `alg`, `kid`, `typ`)
- **Payload** — base64url-encoded JSON, the actual claims (`sub`, `role`, `exp`, custom fields)
- **Signature** — cryptographic proof that header + payload haven't been tampered with since issuance

Decoded payload example:
```json
{
    "iss": "portswigger",
    "exp": 1648037164,
    "name": "Carlos Montoya",
    "sub": "carlos",
    "role": "blog_author",
    "email": "carlos@carlos-montoya.net",
    "iat": 1516239022
}
```

Header and payload are just base64url — **not encrypted, not tamper-proof by themselves**. Anyone can decode and read them. All the security rests on whether the signature is correctly generated and correctly verified.

**JWT vs JWS vs JWE:** the JWT spec only defines the claims format. JWS (JSON Web Signature) defines signed tokens; JWE (JSON Web Encryption) defines encrypted tokens. In practice, "JWT" almost always means a JWS token — encoded and signed, not encrypted.

---

## Why JWT Vulnerabilities Arise

The specs are flexible by design, leaving many implementation details up to the developer. This flexibility is where things go wrong — even with well-established libraries, a developer can misuse the API in a way that quietly breaks verification. Two failure modes dominate:

1. **The signature isn't actually verified properly** — tampering goes undetected
2. **The signature is verified correctly, but the key backing it is guessable, leaked, or reusable in the wrong context**

---

## Attack 1: Accepting Arbitrary Signatures

Libraries typically expose two separate methods: one that **verifies** signatures, one that just **decodes** without checking anything (e.g. Node's `jsonwebtoken` has `verify()` and `decode()`).

If a developer accidentally calls `decode()` on incoming tokens instead of `verify()`, the application never checks the signature at all — any payload, any signature (even garbage), gets accepted as-is.

**Fix:** always use the library's actual verification method, never the decode-only convenience method, on untrusted input.

---

## Attack 2: Accepting Tokens With No Signature (`alg: none`)

The header's `alg` parameter tells the server which algorithm to use for verification — but this value comes from the token itself, which is attacker-controlled and hasn't been verified yet at the point it's read.

JWTs can declare `"alg": "none"`, meaning "unsecured, no signature." Servers are supposed to reject this outright — but if the check is naive string matching, it can sometimes be bypassed with:
- Mixed capitalization: `None`, `NONE`, `nOnE`
- Unexpected encodings

**Note:** even with `alg: none`, the payload must still be terminated with a trailing dot (empty signature segment).

**Root cause:** trusting an attacker-controlled field (`alg`) to decide how much trust to place in the rest of the token.

---

## Attack 3: Brute-Forcing Weak HMAC Secrets

Symmetric algorithms like HS256 sign using a plain secret string — no different from a password. If that secret is weak, default, or copy-pasted from example code and never changed, it can be brute-forced offline with a wordlist (tools like `hashcat` support this directly against JWTs).

Once the secret is recovered, an attacker can sign arbitrary tokens with any claims they want.

**Fix:** long, random, unique HMAC secrets — never hardcoded defaults from documentation/tutorials.

---

## Attack 4: JWT Header Parameter Injection

### `jwk` — Embedded Public Key

The `jwk` header parameter lets the token carry its own public key inline for verification. If the server trusts whatever key is embedded in the token rather than checking it against a known/trusted key, an attacker can:
1. Generate their own RSA keypair
2. Sign a forged token with their own private key
3. Embed their own public key in the `jwk` header
4. Server "verifies" the signature against the attacker's own public key — of course it matches, since the attacker controls both halves

**Fix:** never trust a key embedded in the token itself; verify against a fixed, pre-registered key set (JWKS) — and even then, whitelist expected `kid`s.

### `jku` — JWK Set URL

Same idea, but the header points to a URL hosting the key set (`jku`) instead of embedding the key directly. If the server fetches and trusts whatever JWKS is at that URL without validating the domain, an attacker hosts their own JWKS and points `jku` at it.

**Fix:** whitelist trusted `jku` domains server-side; never blindly fetch attacker-supplied URLs.

### `kid` — Key ID Path Traversal

`kid` tells the server which key (from a set) to use for verification. The spec leaves `kid` as an arbitrary string — some implementations use it as a lookup key into a filesystem path or database.

If `kid` is vulnerable to path traversal, an attacker can point it at an arbitrary file on the server's filesystem, forcing that file's contents to be used as the verification key.

This is only exploitable if:
1. `kid` resolves to a file path without sanitization, **and**
2. The server also accepts a symmetric algorithm (HS256), so the attacker can *know* the "key" value in advance (rather than needing to guess an asymmetric private key)

`/dev/null` is the classic target — it exists on virtually all Linux systems and is guaranteed to be empty (0 bytes). An attacker:
1. Sets `kid` to a traversal path resolving to `/dev/null`
2. Sets `alg: HS256`
3. Signs the forged token using an **empty string** as the HMAC key
4. Server reads `/dev/null` (empty), uses that as the HMAC key, computes the same signature → matches

**Fix:** never use unsanitized user input to build filesystem paths; whitelist acceptable `kid` values.

---

## Attack 5: Algorithm Confusion (RS256 → HS256)

The most conceptually interesting JWT attack. Arises from a mismatch between what a **generic verify function** does and what the **developer assumed** it would only ever be used for.

### The vulnerable pattern

Many libraries expose one verify method that dispatches behavior based on the token's own `alg` header:

```
function verify(token, secretOrPublicKey){
    algorithm = token.getAlgHeader();
    if (algorithm == "RS256") {
        // treat key as an RSA public key (asymmetric verify)
    } else if (algorithm == "HS256") {
        // treat key as an HMAC secret (symmetric verify)
    }
}
```

A developer who only ever intends to use RS256 might write:

```
publicKey = <public-key-of-server>;
verify(token, publicKey);
```

This looks safe — a public key is, by design, safe to share. But nothing stops an attacker from setting `alg: HS256` in their own forged token. When that happens, the *same* generic verify method reinterprets the public key bytes as an **HMAC secret** instead of an RSA public key — and HMAC doesn't care what the key was "supposed" to mean, it just hashes with whatever bytes it's given.

Since the public key is, by definition, not secret, the attacker already knows its value. They sign their own forged token with `HMAC-SHA256(key = public_key_bytes, message = header.payload)`, and the server — now doing symmetric verification against that same public key — accepts it.

**Root cause, one sentence:** the server lets attacker-controlled input (`alg`) decide which trust model applies to a fixed key value, collapsing "safe to share" (asymmetric public key) and "must stay secret" (symmetric key) into the same variable.

### Performing the attack

1. **Obtain the server's public key** — often exposed at `/jwks.json` or `/.well-known/jwks.json` as a JWK Set:
   ```json
   {
       "keys": [
           {
               "kty": "RSA",
               "e": "AQAB",
               "kid": "75d0ef47-af89-47a9-9061-7c02a610d5ab",
               "n": "o-yy1wpYmffgXBxhAUJzHHocCuJolwDqql75ZWuCQ_cb33K2vh9mk6GPM9gNN4Y_qTVX67WhsN3JvaFYw-fhvsWQ"
           }
       ]
   }
   ```

2. **Convert the public key to the exact format the server uses internally.** Every byte must match, including line-ending style (`\n` vs `\r\n`) — HMAC is byte-exact, so even invisible formatting differences produce a completely different signature. In Burp with the JWT Editor extension:
   - JWT Editor Keys tab → New RSA Key → paste the JWK → OK
   - Right-click the key → Copy Public Key as PEM
   - Decoder tab → Base64-encode the PEM (standard Base64, not URL-safe)
   - JWT Editor Keys tab → New Symmetric Key → Generate (placeholder)
   - Replace the `k` value with the Base64-encoded PEM from above
   - Save

3. **Modify the JWT** — set `alg: HS256`, change whatever claims are needed (e.g. `sub: administrator`)

4. **Sign** using the symmetric key (i.e. the public key reinterpreted as an HMAC secret), send the request

If the format doesn't match exactly, the signature won't verify — expect to experiment with PEM variants (with/without trailing newline, full PEM vs body-only, PKCS1 vs X.509) if the first attempt fails.

### When the public key isn't exposed: deriving `n` from existing tokens

If there's no JWKS endpoint, the RSA modulus `n` can sometimes be **mathematically derived** from two valid signed JWTs issued by the same key — no brute force involved.

**Why this is possible:** RSA verification checks `signature^e mod n == derived_value(message)`. Rearranged, this means `n` divides `(signature^e - derived_value(message))` for any validly signed message. With two different signed messages from the same key, you get two such quantities that `n` divides into — computing their **GCD** isolates `n` (or a small multiple of it), since it's astronomically unlikely anything else divides both.

This doesn't break RSA or recover the private key — `n` is meant to be public. It just means you don't need the server to hand it to you directly.

**Tooling:**
```bash
docker run --rm -it portswigger/sig2n <token1> <token2>
```
(or the standalone `jwt_forgery.py` script from the `rsa_sign2n` GitHub repo)

This outputs one or more **candidate** `n` values (the GCD math doesn't always converge to a single unique answer), each packaged as:
- A Base64-encoded PEM key (X.509 and PKCS1 variants)
- A pre-forged test JWT signed using that candidate as the HMAC key, with the original claims unchanged

Send each test JWT to the server — only the correct candidate's token will be accepted as a normal valid session. Once identified, use that key to build the full algorithm confusion attack exactly as above (forge `sub: administrator`, sign, send).

**Fix for algorithm confusion generally:**
- Pin the expected algorithm server-side; reject tokens whose `alg` doesn't match, rather than trusting the token to declare it
- Never reuse the same key material across both symmetric and asymmetric code paths

---

## Why These Work — Common Thread

Across every JWT attack, the same root cause repeats: **the server trusts attacker-controlled parts of the token (`alg`, `kid`, `jwk`, `jku`) to decide how much trust to place in the rest of the token.** The fix is always the same shape: pin the security-relevant decisions (algorithm, key source, key identity) server-side, and never let the token itself dictate them.

---

## Labs

| Lab | Difficulty | Technique |
|---|---|---|
| JWT authentication bypass via unverified signature | Apprentice | Server calls `decode()` instead of `verify()` |
| JWT authentication bypass via flawed signature verification | Apprentice | `alg: none` bypass via case/encoding tricks |
| JWT authentication bypass via weak signing key | Apprentice | Brute-force HMAC secret with a wordlist |
| JWT authentication bypass via jwk header injection | Practitioner | Embed attacker-controlled public key in `jwk` |
| JWT authentication bypass via jku header injection | Practitioner | Host malicious JWKS, point `jku` at it |
| JWT authentication bypass via kid header path traversal | Practitioner | Traverse `kid` to `/dev/null`, sign with empty key |
| JWT authentication bypass via algorithm confusion | Expert | Sign with server's exposed RSA public key as HMAC secret |
| JWT authentication bypass via algorithm confusion with no exposed key | Expert | Derive `n` from two tokens via GCD, then algorithm confusion |