# File Upload Vulnerabilities

## Overview

File upload vulnerabilities occur when a web server accepts files without properly validating their name, type, contents, or size. What looks like a harmless image upload function can become a vector for remote code execution, stored XSS, or denial of service.

The core danger: if you can upload a file the server will execute, you effectively own the server. A web shell gives you arbitrary command execution over HTTP.

Impact depends on two things: what the server fails to validate, and what it does with the file after upload.

---

## How Servers Handle Uploaded Files

The server parses the file extension, maps it to a MIME type, and decides what to do:

- **Non-executable** (image, HTML) — served as-is to the client
- **Executable + configured to run** (PHP on Apache with PHP module loaded) — executed server-side, output returned
- **Executable + not configured to run** — returned as plain text, which leaks source code but prevents execution

The attack only works if the file lands somewhere the server is configured to execute it. Upload directory vs. static assets directory matters.

---

## Web Shell Basics

A web shell is a server-side script that proxies system commands over HTTP.

```php
<?php echo system($_GET['command']); ?>
```

Once uploaded and accessible via URL:

```
GET /uploads/shell.php?command=id HTTP/1.1
```

The server executes `id` and returns the output. From here: read files, write files, pivot internally, exfiltrate data.

---

## Bypass Techniques

### Content-Type spoofing

The server validates the `Content-Type` header in the multipart form data — not the actual file contents. Change it in Burp Repeater:

```
Content-Disposition: form-data; name="avatar"; filename="shell.php"
Content-Type: image/jpeg        ← spoofed, server trusts it
```

The file is `.php`, the server thinks it's a JPEG, uploads it, executes it.

### Extension blacklist bypass

Blacklists are incomplete by design — there are too many executable extensions to enumerate. Common bypasses:

| Technique | Example |
|-----------|---------|
| Alternative extensions | `.php5`, `.phtml`, `.shtml`, `.php7` |
| Case variation | `shell.pHp` (if check is case-sensitive but execution isn't) |
| Double extension | `shell.php.jpg` (parser takes last extension, or first) |
| Trailing characters | `shell.php.` or `shell.php ` (dot/space stripped post-check) |
| Null byte injection | `shell.php%00.jpg` — C-level functions treat `%00` as end of string |
| Recursive strip bypass | `shell.p.phphp` — if `.php` is stripped non-recursively, result is `shell.php` |
| URL encoding | `shell%2Ephp` — decoded after validation |

### .htaccess / web.config upload

Apache loads `.htaccess` from the directory it's in. If you can upload one, you can configure that directory to execute arbitrary extensions as PHP:

```apache
AddType application/x-httpd-php .jpg
```

Now upload `shell.jpg` — Apache executes it as PHP. No extension blacklist matters.

IIS equivalent: upload a `web.config` with a MIME mapping that makes your extension executable.

### Path traversal on filename

If the server uses the filename directly when saving:

```
Content-Disposition: form-data; name="avatar"; filename="../shell.php"
```

The file lands one directory up from the upload folder — potentially in a directory where execution is configured. May need URL encoding: `..%2Fshell.php`.

### Polyglot file

A file that is simultaneously a valid JPEG and valid PHP. The JPEG magic bytes (`FF D8 FF`) satisfy signature-based validation; the PHP payload executes when the server runs it.

Craft with ExifTool — inject PHP into EXIF metadata:

```bash
exiftool -Comment='<?php echo system($_GET["cmd"]); ?>' image.jpg -o polyglot.php
```

Upload as `polyglot.php`. JPEG signature check passes. Server executes PHP payload.

### Race condition

Some servers upload to a temp location, validate, then delete if invalid — but there's a window between upload and deletion. Steps:

1. Upload malicious file
2. Immediately request it in parallel (Burp Intruder with many concurrent requests)
3. Hit the window where the file exists but hasn't been deleted yet

The file executes in the milliseconds before validation removes it. Extend the window by uploading a large file so processing takes longer.

---

## Without Remote Code Execution

If execution is blocked entirely, file uploads still offer:

- **Stored XSS** — upload an HTML or SVG file containing `<script>` tags. If served from the same origin, executes in victim's browser.
- **XXE via file parsing** — if the server parses uploaded `.docx`, `.xlsx`, or other XML-based formats, inject XXE payloads inside.
- **DoS** — upload files large enough to fill disk space.
- **PUT method** — some servers support `PUT /uploads/shell.php` directly. Test with `OPTIONS` first.

---

## Why These Work

The root cause in every case is the same: **validation and execution happen in different places, by different components, with different rules**.

- The form validates Content-Type. The filesystem executes by extension.
- The blacklist checks `.php`. The server executes `.php5`.
- The validator checks the first extension. The server runs the last.
- The upload folder blocks execution. The path traversal puts the file elsewhere.

The fix is always to close the gap between what's validated and what's executed.

---

## Prevention

- **Whitelist extensions**, not blacklist — define what's allowed, reject everything else
- **Rename files on upload** — strip the original name entirely, generate a random one
- **Validate contents**, not just headers — magic bytes, file dimensions, not just `Content-Type`
- **Store outside webroot** — files that aren't served directly can't be executed via URL
- **Disable execution in upload directories** — even if a PHP file gets through, the directory shouldn't run it
- **Validate before permanent storage** — temp directory first, validate, then move
- **Use an established framework** — don't write your own upload validation

---

## Labs

| Lab | Difficulty | Technique |
|-----|-----------|-----------|
| Remote code execution via web shell upload | Apprentice | No validation — upload `.php` directly |
| Web shell upload via Content-Type restriction bypass | Apprentice | Spoof `Content-Type: image/jpeg` in Burp |
| Web shell upload via path traversal | Practitioner | `../shell.php` in filename field |
| Web shell upload via extension blacklist bypass | Practitioner | Upload `.htaccess` to make `.jpg` executable |
| Web shell upload via obfuscated file extension | Practitioner | Null byte: `shell.php%00.jpg` |
| Remote code execution via polyglot web shell upload | Practitioner | JPEG + PHP via ExifTool EXIF metadata |
| Web shell upload via race condition | Expert | Parallel requests to hit upload-before-delete window |