
| Categories                           | Stage 1 |
| ------------------------------------ | ------- |
| SQL Injection                        | X       |
| Authentication                       | X       |
| Directory Traversal                  |         |
| Command Injection                    |         |
| Business Logic Vulnerabilities       |         |
| Information Disclosure               |         |
| Access Control                       | X       |
| File Upload Vulnerabilities          |         |
| Server-Side Request Forgery (SSRF)   |         |
| XXE Injection                        |         |
| Cross-site Scripting (XSS)           | X       |
| Cross-site Request Forgery (CSRF)    |         |
| Cross-origin Resource Sharing (CORS) | X       |
| Clickjacking                         | X       |
| DOM-based Vulnerabilities            | X       |
| WebSockets                           | X       |
| Insecure Deserialization             |         |
| Server-side Template Injection       |         |
| Web Cache Poisoning                  | X       |
| HTTP Host Header Attacks             | X       |
| HTTP Request Smuggling               | X       |
| OAuth Authentication                 | X       |
| JWT Attacks                          | X       |
| Prototype Pollution                  | X       |

*Checkboxes: `[ ]` tested &nbsp;&nbsp; `[ ]` found*

## Authentication

- [ ] [ ] **Login page**
  - [Username enumeration via different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-different-responses)
  - [Username enumeration via subtly different responses](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-subtly-different-responses)
  - [Username enumeration via response timing](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-response-timing)

- [ ] [ ] **Login page that locks out after X failed attempts** (e.g., account locked after 3 tries)
  - [Broken brute-force protection, IP block](https://portswigger.net/web-security/authentication/password-based/lab-broken-bruteforce-protection-ip-block)

- [ ] [ ] **Login page shows error: "You have made too many incorrect login attempts"**
  - [Username enumeration via account lock](https://portswigger.net/web-security/authentication/password-based/lab-username-enumeration-via-account-lock)

- [ ] [ ] **MFA / 2FA step appears after entering credentials**
  - [2FA simple bypass](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-simple-bypass)
  - [2FA broken logic](https://portswigger.net/web-security/authentication/multi-factor/lab-2fa-broken-logic)

- [ ] [ ] **Forgot password / password reset link functionality**
  - [Password reset broken logic](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-broken-logic)
  - [Password reset poisoning via middleware](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-reset-poisoning-via-middleware)

- [ ] [ ] **`stay-logged-in` cookie issued after login**
  - [Brute-forcing a stay-logged-in cookie](https://portswigger.net/web-security/authentication/other-mechanisms/lab-brute-forcing-a-stay-logged-in-cookie)
  - [Offline password cracking](https://portswigger.net/web-security/authentication/other-mechanisms/lab-offline-password-cracking) *(also requires XSS on the site)*

- [ ] [ ] **Password change function (while logged in) that includes a username field or allows changing the username**
  - [Password brute-force via password change](https://portswigger.net/web-security/authentication/other-mechanisms/lab-password-brute-force-via-password-change)

## SQL Injection

- [ ] [ ] **Login form (username/password fields)**
  - [SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)

- [ ] [ ] **Input field in URL query parameter where results are returned in the response** (e.g., `?category=`, `?filter=`)
  - [SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)
  - [SQL injection UNION attack, determining the number of columns returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns)
  - [SQL injection UNION attack, finding a column containing text](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text)
  - [SQL injection UNION attack, retrieving data from other tables](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables)
  - [SQL injection UNION attack, retrieving multiple values in a single column](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)
  - [SQL injection attack, querying the database type and version on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle) *(Oracle indicators: `dual`, ORA- errors)*
  - [SQL injection attack, querying the database type and version on MySQL and Microsoft](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)
  - [SQL injection attack, listing the database contents on non-Oracle databases](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)
  - [SQL injection attack, listing the database contents on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-oracle)

- [ ] [ ] **Input field with NO visible DB output, but the response differs between TRUE vs FALSE conditions** (content length, text, or an SQL error)
  - [Blind SQL injection with conditional responses](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses)
  - [Blind SQL injection with conditional errors](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors)
  - [Visible error-based SQL injection](https://portswigger.net/web-security/sql-injection/blind/lab-sql-injection-visible-error-based) *(verbose SQL error visible in response)*

- [ ] [ ] **Input field with NO visible output and NO conditional difference** (try timing attacks and out-of-band)
  - [Blind SQL injection with time delays](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays)
  - [Blind SQL injection with time delays and information retrieval](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval)
  - [Blind SQL injection with out-of-band interaction](https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band) *(Burp Collaborator DNS lookup)*
  - [Blind SQL injection with out-of-band data exfiltration](https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band-data-exfiltration)

- [ ] [ ] **XML or JSON POST body** (e.g., stock check, order submission) **with WAF blocking standard SQL payloads**
  - [SQL injection with filter bypass via XML encoding](https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding)

## Access Control

- [ ] [ ] **`/robots.txt` reveals a hidden admin or sensitive endpoint**
  - [Unprotected admin functionality](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality)

- [ ] [ ] **JavaScript source in page reveals a hidden admin path** (e.g., variable like `administratorPanelURL`)
  - [Unprotected admin functionality with unpredictable URL](https://portswigger.net/web-security/access-control/lab-unprotected-admin-functionality-with-unpredictable-url)

- [ ] [ ] **Cookie contains a role or admin flag** (e.g., `Admin=false`, `role=user`)
  - [User role controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-role-controlled-by-request-parameter)

- [ ] [ ] **JSON API response for a profile update exposes a `roleID` or privilege field not shown in the UI**
  - [User role can be modified in user profile](https://portswigger.net/web-security/access-control/lab-user-role-can-be-modified-in-user-profile)

- [ ] [ ] **URL contains a predictable numeric user ID** (e.g., `?id=3`, `/my-account?id=1`)
  - [User ID controlled by request parameter](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter)
  - [User ID controlled by request parameter with data leakage in redirect](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-data-leakage-in-redirect) *(302 redirect body contains user data)*
  - [User ID controlled by request parameter with password disclosure](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-password-disclosure) *(password visible in a form field)*

- [ ] [ ] **URL uses an opaque/GUID user ID, but another area of the app leaks other users' GUIDs** (e.g., forum posts, comments)
  - [User ID controlled by request parameter, with unpredictable user IDs](https://portswigger.net/web-security/access-control/lab-user-id-controlled-by-request-parameter-with-unpredictable-user-ids)

- [ ] [ ] **Downloadable file or document with a sequential filename in URL** (e.g., `/files/transcript/2.txt`)
  - [Insecure direct object references](https://portswigger.net/web-security/access-control/lab-insecure-direct-object-references)

- [ ] [ ] **Admin panel is access-controlled, but `X-Original-URL` or `X-Rewrite-URL` header is accepted by the server**
  - [URL-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-url-based-access-control-can-be-circumvented)

- [ ] [ ] **Admin action only available via POST** (try switching to GET or other HTTP methods)
  - [Method-based access control can be circumvented](https://portswigger.net/web-security/access-control/lab-method-based-access-control-can-be-circumvented)

- [ ] [ ] **Multi-step admin workflow** (e.g., upgrade role: step 1 → confirm step 2)
  - [Multi-step process with no access control on one step](https://portswigger.net/web-security/access-control/lab-multi-step-process-with-no-access-control-on-one-step)

- [ ] [ ] **Admin action validates `Referer` header** (only allows requests that came from the admin panel)
  - [Referer-based access control](https://portswigger.net/web-security/access-control/lab-referer-based-access-control)

## Cross-site Scripting

*For Stage 1: XSS is used to steal session cookies, capture credentials, or perform CSRF (change email → password reset).*

- [ ] [ ] **Comment section, user profile, or stored field displayed to other users (including admins)**
  - [Stored XSS into HTML context with nothing encoded](https://portswigger.net/web-security/cross-site-scripting/stored/lab-html-context-nothing-encoded)
  - [Exploiting cross-site scripting to steal cookies](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-stealing-cookies)
  - [Exploiting cross-site scripting to capture passwords](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-capturing-passwords)
  - [Exploiting XSS to bypass CSRF defenses](https://portswigger.net/web-security/cross-site-scripting/exploiting/lab-perform-csrf) *(also requires a "change email" + forgot password flow on the site)*

- [ ] [ ] **Input parameter reflected in HTML response without encoding** (URL query param, search field)
  - [Reflected XSS into HTML context with nothing encoded](https://portswigger.net/web-security/cross-site-scripting/reflected/lab-html-context-nothing-encoded)

- [ ] [ ] **JavaScript uses `document.write` or reads from `location.search` / `location.hash` to render URL input into the page**
  - [DOM XSS in `document.write` sink using source `location.search`](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-document-write-sink)

- [ ] [ ] **`ng-app` attribute in page source** (AngularJS framework detected)
  - [DOM XSS in AngularJS expression with angle brackets and double quotes HTML-encoded](https://portswigger.net/web-security/cross-site-scripting/dom-based/lab-angularjs-expression)

## Cross-origin Resource Sharing (CORS)

- [ ] [ ] **Response `Access-Control-Allow-Origin` reflects the request `Origin` value + `Access-Control-Allow-Credentials: true`**
  - [CORS vulnerability with basic origin reflection](https://portswigger.net/web-security/cors/lab-basic-origin-reflection-attack)

- [ ] [ ] **Response has `Access-Control-Allow-Origin: null` + `Access-Control-Allow-Credentials: true`**
  - [CORS vulnerability with trusted null origin](https://portswigger.net/web-security/cors)

- [ ] [ ] **`Access-Control-Allow-Origin` whitelists a specific subdomain of the app + that subdomain has an XSS vulnerability**
  - [CORS vulnerability with trusted insecure protocols](https://portswigger.net/web-security/cors)

## Clickjacking

- [ ] [ ] **No `X-Frame-Options` header and no `Content-Security-Policy: frame-ancestors` directive in response**
  - [Basic clickjacking with CSRF token protection](https://portswigger.net/web-security/clickjacking)
  - [Clickjacking with form input data prefilled from URL parameter](https://portswigger.net/web-security/clickjacking) *(also requires a POST form prefillable via GET query params)*
  - [Multistep clickjacking](https://portswigger.net/web-security/clickjacking) *(also requires a multi-step confirmation action on the site)*

- [ ] [ ] **Frame-busting JavaScript present but no `X-Frame-Options` header** (neutralise with `sandbox="allow-forms"` on the iframe)
  - [Exploiting clickjacking vulnerability to trigger DOM-based XSS](https://portswigger.net/web-security/clickjacking)

## DOM-based Vulnerabilities

- [ ] [ ] **JavaScript source has `window.addEventListener('message', ...)` that passes `e.data` to an `innerHTML` sink**
  - [DOM XSS using web messages](https://portswigger.net/web-security/dom-based)

- [ ] [ ] **`postMessage` listener passes URL to a `location.href` sink, validating only that it starts with `http:`**
  - [DOM XSS using web messages and a JavaScript URL](https://portswigger.net/web-security/dom-based)

- [ ] [ ] **`postMessage` listener calls `JSON.parse(e.data)` and sets `iframe.src` from the parsed `url` property**
  - [DOM XSS using web messages via a JSON.parse() call](https://portswigger.net/web-security/dom-based)

- [ ] [ ] **URL parameter** (e.g., `?url=`, `?redirect=`) **passed directly to `location.href` without full-domain validation**
  - [DOM-based open redirection](https://portswigger.net/web-security/dom-based)

- [ ] [ ] **`document.cookie` stores the current `window.location` value, and that cookie is reflected in an HTML attribute**
  - [DOM-based cookie manipulation](https://portswigger.net/web-security/dom-based)

## WebSockets

- [ ] [ ] **WebSocket connections visible in Burp's WebSockets history tab + user input reflected back without encoding**
  - [Manipulating WebSocket messages to exploit vulnerabilities](https://portswigger.net/web-security/websockets/lab-manipulating-messages-to-exploit-vulnerabilities)
  - [Manipulating the WebSocket handshake to exploit vulnerabilities](https://portswigger.net/web-security/websockets) *(blacklist present — try case variation, backticks, HTML encoding)*

- [ ] [ ] **WebSocket handshake request uses only session cookie with no unpredictable CSRF-like token**
  - [Cross-site WebSocket hijacking](https://portswigger.net/web-security/websockets)

## Web Cache Poisoning

- [ ] [ ] **Response includes caching headers (`X-Cache`, `Age`) + `X-Forwarded-Host` reflected in dynamically generated JS `src` or redirect URL**
  - [Web cache poisoning with an unkeyed header](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-an-unkeyed-header)
  - [Web cache poisoning with multiple headers](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-multiple-headers) *(also requires HTTP→HTTPS redirect and a static JS file endpoint)*

- [ ] [ ] **An unkeyed cookie value reflected unsafely inside a `<script>` block in the response**
  - [Web cache poisoning with an unkeyed cookie](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-with-an-unkeyed-cookie)

- [ ] [ ] **Response has `Vary: User-Agent` + Param Miner finds an unknown unkeyed header reflected in a JS `src` attribute**
  - [Targeted web cache poisoning using an unknown header](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-targeting-an-unknown-header)

- [ ] [ ] **DOM XSS gadget exists in a cached JS file + unkeyed response header can inject a domain value into it**
  - [Web cache poisoning to exploit a DOM vulnerability via a cache with strict cacheability criteria](https://portswigger.net/web-security/web-cache-poisoning/exploiting-design-flaws/lab-web-cache-poisoning-to-exploit-a-dom-vulnerability-via-a-cache-with-strict-cacheability-criteria)

- [ ] [ ] **URL query string excluded from the cache key but its value is reflected in the response** (Param Miner: "Unkeyed param")
  - [Web cache poisoning via an unkeyed query string](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-unkeyed-query-string)

- [ ] [ ] **Specific tracking/analytics parameter excluded from cache key but reflected** (Param Miner: "Guess GET Parameters")
  - [Web cache poisoning via an unkeyed query parameter](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-unkeyed-query)

- [ ] [ ] **Parsing discrepancy between cache and backend** — cache strips a parameter the app still processes
  - [Web cache poisoning via parameter cloaking](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-parameter-cloaking)

- [ ] [ ] **App accepts GET requests with a body + body parameter not included in the cache key**
  - [Web cache poisoning via a fat GET request](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-fat-get)

- [ ] [ ] **Unencoded XSS payload in URL path reflected in a cached response** (browser normally encodes it — send via Burp directly)
  - [URL normalization](https://portswigger.net/web-security/web-cache-poisoning/exploiting-implementation-flaws/lab-web-cache-poisoning-url-normalization)

## HTTP Host Header Attacks

- [ ] [ ] **Forgot password functionality exists + reset link URL is built using the Host header value**
  - [Password reset poisoning via middleware](https://portswigger.net/web-security/host-header) *(also see Authentication section)*

- [ ] [ ] **Admin panel at `/admin` restricted to `localhost` or internal IP**
  - [Host header authentication bypass](https://portswigger.net/web-security/host-header/exploiting/lab-host-header-authentication-bypass)

- [ ] [ ] **Duplicate `Host:` header accepted + injected value reflected in HTML or JS src without encoding**
  - [Web cache poisoning via ambiguous requests](https://portswigger.net/web-security/host-header)

- [ ] [ ] **Burp Collaborator receives a DNS lookup when the Host header is modified**
  - [Routing-based SSRF](https://portswigger.net/web-security/host-header)

- [ ] [ ] **Server accepts an absolute URL in the request line** (e.g., `GET https://victim.com/ HTTP/1.1`) while Host contains a different value
  - [SSRF via flawed request parsing](https://portswigger.net/web-security/host-header)

## HTTP Request Smuggling

- [ ] [ ] **App is behind a reverse proxy, CDN, or load balancer** — probe for CL.TE / TE.CL / TE.TE
  - [HTTP request smuggling, basic CL.TE vulnerability](https://portswigger.net/web-security/request-smuggling/lab-basic-cl-te) *(timeout on CL+TE probe — front-end uses CL)*
  - [HTTP request smuggling, basic TE.CL vulnerability](https://portswigger.net/web-security/request-smuggling/lab-basic-te-cl) *(differential response — back-end uses CL)*
  - [HTTP request smuggling, obfuscating the TE header](https://portswigger.net/web-security/request-smuggling) *(both servers support TE — try obfuscated TE value)*

- [ ] [ ] **Smuggling confirmed + admin panel at `/admin` only accessible from `localhost`**
  - [Exploiting HTTP request smuggling to bypass front-end security controls, TE.CL vulnerability](https://portswigger.net/web-security/request-smuggling/exploiting/lab-bypass-front-end-controls-te-cl)

- [ ] [ ] **Smuggling confirmed + app stores user-submitted content** (comments, messages, search history)
  - [Exploiting HTTP request smuggling to capture other users' requests](https://portswigger.net/web-security/request-smuggling/exploiting/lab-capture-other-users-requests)

## OAuth Authentication

- [ ] [ ] **"Sign in with [provider]" social login button on login/register page**
  - [Authentication bypass via OAuth implicit flow](https://portswigger.net/web-security/oauth/lab-oauth-authentication-bypass-via-oauth-implicit-flow) *(implicit flow — token used directly, email param is tamperable)*

- [ ] [ ] **Account linking feature ("Connect your social account") + no `state` parameter in the OAuth authorization request**
  - [Forced OAuth profile linking](https://portswigger.net/web-security/oauth)

- [ ] [ ] **`redirect_uri` parameter in OAuth URL + no strict whitelist validation on the redirect domain**
  - [OAuth account hijacking via redirect_uri](https://portswigger.net/web-security/oauth)

- [ ] [ ] **OAuth implicit flow (access token in URL `#` fragment) + open redirect vulnerability exists in the client app**
  - [Stealing OAuth access tokens via an open redirect](https://portswigger.net/web-security/oauth)

- [ ] [ ] **`/.well-known/openid-configuration` accessible + dynamic client registration endpoint exists (unauthenticated)**
  - [SSRF via OpenID dynamic client registration](https://portswigger.net/web-security/oauth/openid/lab-oauth-ssrf-via-openid-dynamic-client-registration)

- [ ] [ ] **OAuth proxy page in client app relays tokens via `window.postMessage`**
  - [Stealing OAuth access tokens via a proxy page](https://portswigger.net/web-security/oauth/lab-oauth-stealing-oauth-access-tokens-via-a-proxy-page)

## JWT Attacks

- [ ] [ ] **JWT token present** (`Authorization: Bearer eyJ...` or a cookie with three `.`-separated base64 segments)
  - [JWT authentication bypass via unverified signature](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-unverified-signature) *(always try first — modify payload, keep original signature)*
  - [JWT authentication bypass via flawed signature verification](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-flawed-signature-verification) *(try setting `alg: none` / `alg: NoNe`)*

- [ ] [ ] **JWT with `alg: HS256`** (symmetric shared secret — try brute forcing)
  - [JWT authentication bypass via weak signing key](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-weak-signing-secret) *(hashcat + JWT wordlist)*

- [ ] [ ] **JWT with `alg: RS256` + `jwk` parameter present in the JWT header**
  - [JWT authentication bypass via jwk header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jwk-header-injection)

- [ ] [ ] **JWT with `alg: RS256` + `jku` parameter present in the JWT header**
  - [JWT authentication bypass via jku header injection](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-jku-header-injection)

- [ ] [ ] **JWT with `kid` parameter in header** (server reads signing key from a file path based on `kid` value)
  - [JWT authentication bypass via kid header path traversal](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-kid-header-path-traversal)

- [ ] [ ] **JWT with `alg: RS256` + public key exposed** (e.g., `/.well-known/jwks.json`, `/jwks.json`, `/public.pem`)
  - [JWT authentication bypass via algorithm confusion](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-algorithm-confusion)

- [ ] [ ] **JWT with `alg: RS256` + public key NOT exposed** (derive from two token samples with `rsa_sign2n`)
  - [JWT authentication bypass via algorithm confusion with no exposed key](https://portswigger.net/web-security/jwt/lab-jwt-authentication-bypass-via-algorithm-confusion-with-no-exposed-key)

## Prototype Pollution

- [ ] [ ] **Client-side — URL query string accepts `?__proto__[foo]=bar` without error + JS reads from `Object.prototype` in a dangerous sink** (e.g., script `src`)
  - [Client-side prototype pollution via flawed sanitization](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-client-side-prototype-pollution-via-flawed-sanitization)

- [ ] [ ] **Client-side — page uses third-party JS libraries (jQuery, Lodash, etc.) + DOM Invader detects a pollution source**
  - [Client-side prototype pollution in third-party libraries](https://portswigger.net/web-security/prototype-pollution/client-side/lab-prototype-pollution-client-side-prototype-pollution-in-third-party-libraries)

- [ ] [ ] **Server-side — JSON body accepted + JSON response includes `isAdmin: false`, `role:`, or `permissions:` property**
  - [Privilege escalation via server-side prototype pollution](https://portswigger.net/web-security/prototype-pollution)
  - [Detecting server-side prototype pollution without polluted property reflection](https://portswigger.net/web-security/prototype-pollution) *(inject `"__proto__": {"json spaces": 10}` — formatting change in response confirms pollution)*
  - [Remote code execution via server-side prototype pollution](https://portswigger.net/web-security/prototype-pollution) *(also requires Node.js — look for `X-Powered-By: Express` header)*
