
## [DOM XSS using web messages](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages)

- Open the lab and inspect the home page source. Find the `addEventListener('message', ...)` call that writes `e.data` into `innerHTML` with no origin check.
- Go to the **Exploit Server** in the lab interface.
- Paste the following into the **Body** field, replacing the lab ID with your own:

```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">
```

💡 The `onload` fires once the iframe loads, sending a `postMessage` to the vulnerable page. The `*` means any origin is accepted — matching the lack of origin validation in the listener.

- Click **Store**.
- Click **Deliver exploit to victim**.
- 💡 The iframe loads the lab page → `postMessage` sends the `<img>` payload → the listener writes it into `innerHTML` → the broken `src` triggers `onerror` → `print()` is called → lab solved. ✅



## [DOM XSS using web messages and a JavaScript URL](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages-and-a-javascript-url)

- Open the lab and inspect the home page. Find the `addEventListener('message', ...)` call — it checks for `http:`/`https:` but not `javascript:`, and passes the value to `location.href`.
- Go to the **Exploit Server**.
- Paste the following into the **Body**, replacing the lab ID with yours:

```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:print()//http:','*')">
```

💡 `//http:` at the end is a comment that satisfies neither `indexOf` check being `-1` — wait, actually it makes `indexOf('http:')` return a positive value... let's be precise:

- The payload `javascript:print()//https:` works because `indexOf('https:')` finds `https:` in the comment portion, so the `if` condition is satisfied and `location.href` gets set to `javascript:print()` — executing it. ✅
- Click **Store**, then **Deliver exploit to victim**.
- 💡 Flow: iframe loads → `postMessage` fires → listener finds `https:` in the string → assigns full value to `location.href` → `javascript:` executes `print()` → lab solved. ✅



## [DOM XSS using web messages and `JSON.parse`](https://portswigger.net/web-security/dom-based/controlling-the-web-message-source/lab-dom-xss-using-web-messages-and-json-parse)

- Open the lab and inspect the source. Find the `addEventListener('message', ...)` — it parses `e.data` as JSON and sets `iframe.src = d.url` on `load-channel` with no origin validation.
- Go to the **Exploit Server**.
- Paste the following into the **Body**, replacing the lab ID with yours:

html

```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
```

💡 The payload must be valid JSON with `type` set to `"load-channel"` to hit the right switch case, and `url` set to `javascript:print()`.

- Click **Store**, then **Deliver exploit to victim**.
- 💡 Flow: iframe loads the lab → `postMessage` sends JSON → listener parses it → `load-channel` case sets `iframe.src` to `javascript:print()` → `print()` executes → lab solved. ✅

The other cases (`player-height-changed`) could also be abused for CSS injection via `d.width`/`d.height` if there's no numeric validation, but the `javascript:` sink is the cleanest path here.

## [DOM-based open redirection](https://portswigger.net/web-security/dom-based/open-redirection/lab-dom-open-redirection)

- Open the lab and navigate to any blog post page.
- Open browser DevTools (`F12`) and inspect the **"Back to Blog"** link — or simply view the page source.
- Locate the vulnerable `onclick` handler on the anchor tag:

```
  returnUrl = /url=(https?:\/\/.+)/.exec(location);
  if(returnUrl) location.href = returnUrl[1];
```

💡 The script reads a `url=` parameter directly from the page's URL and passes it to `location.href` with no domain validation — the source is `location`, the sink is `location.href`.

- Grab your **Lab ID** and **Exploit Server ID** from the lab interface.
- Construct the malicious URL:

```
  https://YOUR-LAB-ID.web-security-academy.net/post?postId=4&url=https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/
```

- Visit the crafted URL in your browser. 💡 The `url=` parameter matches the regex `https?:\/\/.+`, so the script extracts your exploit server URL and assigns it to `location.href` when "Back to Blog" is clicked.
- Click the **"Back to Blog"** link — your browser will redirect to the exploit server.
- The lab is solved once the redirect to your exploit server fires successfully.

## [DOM-based cookie manipulation](https://portswigger.net/web-security/dom-based/cookie-manipulation/lab-dom-cookie-manipulation)

- In Burp Repeater, find the `GET /` request. Observe `lastViewedProduct` cookie is reflected into an `<a href>` in the response.
- Change the `lastViewedProduct` cookie value to `javascript:print()`.
- Send the request and confirm the response contains `<a href='javascript:print()'>`.
- Go to the **Exploit Server** and deliver a page that sets the cookie then redirects the victim to the lab:


```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&'><script>print()</script>" onload="if(!window.x)this.src='https://YOUR-LAB-ID.web-security-academy.net';window.x=1;">
```

💡 Alternatively, use `document.cookie` to set the payload cookie then redirect.

- The victim visits the home page → clicks "Last viewed product" → `javascript:print()` executes → lab solved. ✅

