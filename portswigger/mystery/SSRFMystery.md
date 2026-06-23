# PortSwigger Mystery Labs — SSRF Checklist

> Use this checklist to solve any SSRF mystery lab. First, identify the lab type, then follow the matching steps below.

---

## Identifying the Lab Type

**Does the stock check request go directly to an internal URL with no filtering?**

- Localhost accessible → **Basic SSRF against local server**
- Internal IP range accessible → **Basic SSRF against back-end system**

**Is the injection point the `Referer` header rather than a body parameter?** → **Blind SSRF with out-of-band detection**

**Is there a filter blocking `localhost` or `127.0.0.1`?** → **SSRF with blacklist-based input filter**

**Are direct requests to other hosts blocked, but the app has a redirect feature?** → **SSRF with filter bypass via open redirection**

> **Recon tip:** Always check every parameter that accepts a URL — `stockApi` is the most common injection point in these labs, but also check headers like `Referer`. Test with a Burp Collaborator URL first to confirm out-of-band callbacks before probing internal hosts.

---

## Lab 1 — Basic SSRF Against the Local Server

- [ ] Browse to a product page, click **Check stock**, and intercept the request in Burp.
- [ ] Send the stock check request to Repeater.
- [ ] Change the `stockApi` parameter value to `http://localhost/admin`.
- [ ] Send the request and confirm the admin panel is returned in the response.
- [ ] Read the HTML to find the delete URL: `http://localhost/admin/delete?username=carlos`.
- [ ] Set `stockApi` to that delete URL and send — confirm carlos is deleted. Lab solved.

**Key concept:** The server-side request is made from the server itself, so `localhost` resolves to the internal admin interface that is not exposed externally.

---

## Lab 2 — Basic SSRF Against Another Back-End System

- [ ] Browse to a product page, click **Check stock**, and intercept the request in Burp.
- [ ] Send the stock check request to **Intruder**.
- [ ] Change the `stockApi` parameter to `http://192.168.0.1:8080/admin`, then highlight the final octet (`1`) and click **Add §**.
- [ ] In the Payloads panel, set payload type to **Numbers**, from `1` to `255`, step `1`.
- [ ] Start the attack, then sort by the **Status** column — find the single `200` response and note the IP.
- [ ] Send that request to Repeater, change the `stockApi` path to `/admin/delete?username=carlos` and send. Lab solved.

**Key concept:** The admin panel lives on an internal back-end host in the `192.168.0.x` subnet. Intruder brute-forces the final octet to find which host is running it.

---

## Lab 3 — Blind SSRF With Out-of-Band Detection

- [ ] Open the lab and browse to any product page.
- [ ] Turn on Burp **Intercept** and reload the product page to capture the `GET` request.
- [ ] Send the request to Repeater.
- [ ] Open the **Collaborator** tab and click **Copy to clipboard** to grab your unique subdomain.
- [ ] In Repeater, find the `Referer` header and replace its entire value with your Collaborator URL:
    
    ```
    Referer: https://YOUR-COLLABORATOR-SUBDOMAIN
    ```
    
- [ ] Send the request.
- [ ] Switch to the Collaborator tab, click **Poll now** — wait a few seconds and retry if nothing shows up yet.
- [ ] Confirm you see both DNS and HTTP interactions coming in from the server. Lab solved.

**Key concept:** The app fetches the `Referer` URL server-side but returns no output. Burp Collaborator acts as the out-of-band listener — DNS and HTTP callbacks confirm the blind SSRF.

---

## Lab 4 — SSRF With Blacklist-Based Input Filter

- [ ] Browse to a product page, click **Check stock**, and intercept the request in Burp.
- [ ] Send the stock check request to Repeater.
- [ ] Change `stockApi` to `http://127.0.0.1/` — confirm it is blocked.
- [ ] Bypass the localhost block by using `http://127.1/` instead — confirm it is allowed.
- [ ] Change to `http://127.1/admin` — confirm the word `admin` is blocked.
- [ ] Bypass the admin block by double-URL encoding the `a`: `http://127.1/%2561dmin` — confirm the admin panel loads.
- [ ] Read the HTML to find the delete URL, then set `stockApi` to `http://127.1/%2561dmin/delete?username=carlos` and send. Lab solved.

**Key concept:** Two separate bypasses are needed — `127.1` is a valid shorthand for `127.0.0.1`, and double-encoding (`a` → `%61` → `%2561`) causes the filter to miss the word `admin` while the back end decodes it correctly.

---

## Lab 5 — SSRF With Filter Bypass via Open Redirection

- [ ] Browse to a product page, click **Check stock**, and intercept the request in Burp.
- [ ] Send the stock check request to Repeater.
- [ ] Try changing `stockApi` to an external host — confirm direct requests to other hosts are blocked.
- [ ] Click **Next product** on any product page and intercept the request — note the `path` parameter is reflected directly into the `Location` redirect header, confirming an open redirect.
- [ ] Back in the stock check Repeater tab, set `stockApi` to:
    
    ```
    /product/nextProduct?path=http://192.168.0.12:8080/admin
    ```
    
- [ ] Confirm the admin panel is returned in the response.
- [ ] Change the path to:
    
    ```
    /product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos
    ```
    
- [ ] Send to solve the lab.

**Key concept:** The filter only blocks external URLs directly in `stockApi`. By pointing it at a trusted internal path that openly redirects to any `path` value, the SSRF filter is bypassed — the server follows the redirect to the internal admin host.

---

## Quick Reference — Payloads

|Lab|Injection Point|Key Payload|
|---|---|---|
|Basic — local server|`stockApi`|`http://localhost/admin`|
|Basic — back-end system|`stockApi`|`http://192.168.0.§1§:8080/admin`|
|Blind OOB|`Referer` header|`https://YOUR-BURP-COLLAB`|
|Blacklist bypass|`stockApi`|`http://127.1/%2561dmin`|
|Open redirect bypass|`stockApi`|`/product/nextProduct?path=http://192.168.0.12:8080/admin`|

---

## Common Delete URL Pattern

Once you have access to the admin panel, the delete endpoint is almost always:

```
<base-admin-url>/delete?username=carlos
```

Swap the base for whatever internal address you discovered in the specific lab.