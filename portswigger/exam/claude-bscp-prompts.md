# Claude Code Prompts — Burp Suite Practitioner Exam

---

## Stage 1

### How to use
1. Export your Burp Suite HTTP history: **Proxy → HTTP history → right-click → Save items** (XML format)
2. Start a Claude Code session from this directory
3. Paste the prompt below, replacing `[PATH_TO_XML]` with the path to your exported file

---

### Prompt

```
You are a professional web penetration tester preparing for the Burp Suite Practitioner exam. Your goal for Stage 1 is to compromise any user account on the target application.

## Step 1 — Load the recon reference

Read the file at: Exam-Guides/Stage 1 Recon.json

This JSON maps observable site features to specific vulnerability categories and PortSwigger labs. Use it as your ground truth for what anomalies to look for and what they mean.

## Step 2 — Analyze the Burp traffic

Read and analyze the Burp Suite HTTP history export at: [PATH_TO_XML]

The XML contains <request> and <response> elements that are base64-encoded.
Decode them all and examine the full raw HTTP traffic.

## Step 3 — Inventory the application

Before looking for vulnerabilities, build a picture of the application:
- All unique endpoints and HTTP methods observed
- Authentication mechanisms present (login form, MFA, OAuth, JWT, cookies)
- Technologies detected (frameworks, server headers, client-side libraries)
- All cookies set and their attributes (HttpOnly, Secure, SameSite, value format)
- All notable response headers

## Step 4 — Hunt for anomalies

For every feature listed in Stage 1 Recon.json, check whether evidence of that feature exists anywhere in the captured traffic. Be thorough — check requests AND responses.

Specifically examine:

**Headers (responses)**
- Access-Control-Allow-Origin, Access-Control-Allow-Credentials
- X-Frame-Options, Content-Security-Policy (frame-ancestors)
- X-Cache, Age, Vary, Cache-Control
- X-Powered-By, Server
- Set-Cookie (flag attributes, suspicious names, encoded values)

**Cookies (requests)**
- Role or admin flags (Admin=, role=, isAdmin=)
- stay-logged-in tokens (check if base64 or hash-based)
- JWT tokens (three base64 segments separated by dots — decode and check alg, kid, jwk, jku fields)

**URLs and parameters**
- Predictable numeric IDs (?id=, ?userId=, ?orderId=)
- GUIDs in URLs — cross-reference with other endpoints that might expose other users' GUIDs
- Sequential filenames in download paths
- /robots.txt content
- Hidden admin paths referenced in JavaScript source

**HTML and JavaScript source (in responses)**
- ng-app attribute (AngularJS)
- document.write() with user-controlled input
- window.addEventListener('message', ...) listeners and what sink they write to
- postMessage usage
- OAuth state parameter presence or absence in authorization URLs
- /.well-known/openid-configuration or dynamic client registration endpoints

**Request patterns**
- WebSocket upgrade requests (Sec-WebSocket-Key, Upgrade: websocket)
- Requests to /admin or restricted paths and their response codes
- Password reset / forgot password flows — does the reset link domain come from the Host header?
- Multi-factor authentication steps
- OAuth flows — note redirect_uri values and whether state parameter is present
- Multi-step workflows (e.g., confirm dialogs) where later steps may lack access control

**Caching behavior**
- Responses with X-Cache or Age headers
- Whether X-Forwarded-Host or other unkeyed headers appear to affect response content
- Whether query string parameters are reflected in cached responses

## Step 5 — Produce the report

Structure your output exactly as follows:

---

### Application Overview
Brief summary: what the app appears to do, technologies identified, authentication mechanisms present.

---

### Anomalies Found

For each anomaly, use this format:

#### [Short anomaly title]
- **Observed**: [Exactly what was seen — include the specific header, parameter, value, or behavior]
- **Location**: [HTTP method + URL where it was observed]
- **Matches feature**: [Copy the exact feature string from Stage 1 Recon.json]
- **Vulnerability category**: [Category name from the JSON]
- **Labs to attempt**:
  - [Lab Name](url)
  - [Lab Name](url)
- **Notes**: [Any conditions from the JSON, or your own observations about exploitability]

---

### Priority Attack Order

Rank all identified anomalies from most to least likely to result in account compromise.
For each: anomaly title + one sentence explaining why it ranks here.

---

### Coverage Gaps

List any Stage 1 features from the JSON that you could not assess because the traffic did not cover that part of the application (e.g., no password reset flow captured, no admin panel requests visible). These are areas to explore manually in Burp.

---

Be specific and evidence-based. Every anomaly must be traceable to something actually observed in the traffic — do not speculate about features with no evidence in the provided requests and responses.
```
