# Subdomain Enumeration Checklist  
  
> **Goal:** Find every subdomain for each root domain in `verified-domains.txt`. Passive sources run first — no direct target con  
tact. Active brute-force only runs after passive is complete and if the program allows it. Output feeds a prioritized list of sub  
domains worth testing.  
  
---  
  
## Step 1 — Pre-Flight Checks  
  
Do these before running any enumeration. Skipping them wastes time and pollutes results.  
  
### Wildcard DNS Detection  
If wildcard DNS is enabled, any random subdomain resolves — brute-forcing will produce thousands of false positives. Check before  
the active phase:  
```bash  
dig RANDOMSTRING99999.target.com +short  
dig RANDOMSTRING88888.target.com +short  
```  
- Both return the same IP → **wildcard enabled** — disable brute-force (Step 4) for this domain, or use `puredns` which handles w  
ildcards automatically  
- No result → **no wildcard** — safe to brute-force  
  
### Confirm Active Scanning Is Allowed  
- [ ] Re-check program scope page — some programs explicitly prohibit automated DNS brute-forcing  
- [ ] If prohibited → run Steps 2–3 (passive only), skip Step 4  
  
### Resolver Setup  
A reliable resolver list prevents poisoned results and rate-limiting from public resolvers:  
```bash  
# Download a fresh trusted resolver list  
wget https://raw.githubusercontent.com/janmasarik/resolvers/master/resolvers.txt -O resolvers.txt  
```  
  
---  
  
## Step 2 — Passive Enumeration  
  
No direct contact with the target. Run all tools against each root domain from `verified-domains.txt`. Each tool saves to its own  
file.  
  
### Subfinder  
Aggregates 30+ passive sources. Fastest passive tool:  
```bash  
subfinder -d target.com -all -silent -o passive-subfinder.txt  
```  
  
### Amass Passive  
Deeper OSINT mining across certificate logs, APIs, and datasets:  
```bash  
amass enum -passive -d target.com -o passive-amass.txt  
```  
  
### Assetfinder  
Lightweight — good at catching domains others miss:  
```bash  
assetfinder --subs-only target.com > passive-assetfinder.txt  
```  
  
### Chaos  
ProjectDiscovery's proprietary DNS dataset — catches subdomains not in any public source:  
```bash  
chaos -d target.com -silent -o passive-chaos.txt  
```  
  
### Findomain  
Fast multi-API passive scan:  
```bash  
findomain -t target.com -q -u passive-findomain.txt  
```  
  
### crt.sh Certificate Transparency  
Direct CT log query — catches subdomains that never appeared in any DNS dataset:  
```bash  
curl -s "https://crt.sh/?q=%25.target.com&output=json" > passive-crtsh.json
cat passive-crtsh.json | jq -r '.[].name_value' > passive-crtsh.txt  
```  
  
### Historical URLs (Waymore)  
Pulls archived URLs from Wayback Machine and Common Crawl — surfaces long-forgotten subdomains:  
```bash  
waymore -i domains.txt -mode U -oU passive-waymore.txt
cat passive-waymore.txt | cut -d '/' -f3 | sort -u | grep "\.target\.com$" | sort -u > passive-gau-subs.txt  
```  
  
### Wayback CDX API  
Direct Wayback Machine query — overlaps with GAU but sometimes catches additional entries:  
```bash  
cat domains.txt | while read i; do curl -silent 'https://web.archive.org/cdx/search/cdx?url=*.'$i'/*&collapse=urlkey&output=text&fl=original' | anew passive-wayback.txt; done

cat passive-wayback.txt | cut -d '/' -f3 | sort -u | grep "\.target\.com$" | sort -u > passive-gau-subs.txt

curl -s "https://web.archive.org/cdx/search/cdx?url=*.target.com/*&output=text&fl=original&collapse=urlkey" \  
 | unfurl -u domains \  
 | grep "\.target\.com$" \  
 | sort -u > passive-wayback.txt  
```  
  
### Subdomain.center API  
Free aggregated subdomain database:  
```bash  
curl -s "https://api.subdomain.center/?domain=target.com" \  
 | jq -r '.[]' \  
 | sort -u > passive-subdomaincenter.txt  
```  
  
---  
  
## Step 3 — Consolidate Passive Results  
  
Merge all passive output, strip wildcards, normalise case, and filter to only the target domain:  
```bash  
cat passive-*.txt \  
 | sed 's/\*\.//g' \  
 | tr '[:upper:]' '[:lower:]' \  
 | grep "\.target\.com$\|^target\.com$" \  
 | sort -u > passive-all.txt  
  
wc -l passive-all.txt  
```  
  
Resolve passive results to confirm which ones actually have DNS records before moving to active:  
```bash  
dnsx -l passive-all.txt -silent -a -o passive-resolved.txt  
wc -l passive-resolved.txt  
```  
  
---  
  
## Step 4 — Active Brute-Force  
  
**Skip this step if:** wildcard DNS is enabled and you are not using puredns, or the program prohibits active scanning.  
  
`puredns` is preferred over raw `massdns` because it automatically handles wildcard filtering:  
  
### Brute-Force with puredns  
```bash  
puredns bruteforce /path/to/wordlist.txt target.com \  
 -r resolvers.txt \  
 -w active-bruteforce.txt  
```  
  
### Brute-Force with Amass  
```bash  
amass enum -brute \  
 -w /path/to/wordlist.txt \  
 -d target.com \  
 -r resolvers.txt \  
 -o active-amass-brute.txt  
```  
  
Merge active results into the main resolved list:  
```bash  
cat active-bruteforce.txt active-amass-brute.txt \  
 | sort -u >> passive-resolved.txt  
  
sort -u passive-resolved.txt -o all-resolved.txt  
```  
  
---  
  
## Step 5 — Recursive Enumeration  
  
Take interesting subdomains discovered so far and treat them as new targets. Focus on subdomains that suggest nested infrastructure:  
```bash  
# Extract candidates worth recursing into  
grep -E "\.(dev|api|staging|internal|corp|vpn|admin|portal|test|uat)\." all-resolved.txt > recursive-candidates.txt  
```  
  
Run passive + brute-force on each candidate:  
```bash  
while IFS= read -r subdomain; do  
 subfinder -d "$subdomain" -all -silent >> recursive-passive.txt  
 puredns bruteforce /path/to/wordlist.txt "$subdomain" -r resolvers.txt >> recursive-brute.txt  
done < recursive-candidates.txt  
```  
  
Resolve and merge recursive results:  
```bash  
cat recursive-passive.txt recursive-brute.txt \  
 | sort -u \  
 | dnsx -silent -a \  
 >> all-resolved.txt  
  
sort -u all-resolved.txt -o all-resolved.txt  
```  
  
---  
  
## Step 6 — Permutation and Alteration  
  
Generate smart variations of already-discovered subdomains. Finds targets that wordlists never include because they follow the company's own naming patterns:  
  
### dnsgen (pattern-aware permutations)  
```bash  
cat all-resolved.txt | dnsgen - \  
 | massdns -r resolvers.txt -t A -o S \  
 | awk '{print $1}' \  
 | sed 's/\.$//' \   
 | sort -u > permutations-resolved.txt 
```  
  
### gotator (depth-controlled permutation)  
```bash  
gotator -sub all-resolved.txt \  
 -perm /path/to/permutation-wordlist.txt \  
 -depth 1 -silent \  
 | puredns resolve -r resolvers.txt \  
 >> permutations-resolved.txt  
```  
  

### Recommended Wordlists (pick one based on time available)  
| Wordlist | Size | Source |  
|---|---|---|  
| `best-dns-wordlist.txt` | ~9M | Assetnote — `wordlists.assetnote.io` |  
| `subdomains-top1million-110000.txt` | 110K | SecLists |  
| `all.txt` | ~2M | Jason Haddix |  
  

### Amass alteration  
```bash  
amass enum -active -d target.com -alter -o amass-altered.txt  
```  
  
Merge permutation results:  
```bash  
cat permutations-resolved.txt amass-altered.txt \  
 | sort -u >> all-resolved.txt  
  
sort -u all-resolved.txt -o all-resolved.txt  
wc -l all-resolved.txt  
```  
  
---  
  
## Step 7 — HTTP Probing  
  
Filter the resolved list down to subdomains with live web services. This is the primary cut before the assessment step.  
  
### Basic probe — extract status codes and page titles  
```bash  
httpx -l all-resolved.txt \  
 -silent \  
 -status-code \  
 -title \  
 -o httpx-all.txt  
```  
  
### Filter to interesting status codes only  
```bash  
httpx -l all-resolved.txt \  
 -silent \  
 -mc 200,301,302,401,403 \  
 -status-code \  
 -title \  
 -o live-interesting.txt  
```  
  
Drop parked pages and placeholder responses:  
```bash  
httpx -l all-resolved.txt \  
 -silent \  
 -mc 200,301,302,401,403 \  
 -fpt parked \  
 -status-code \  
 -title \  
 -o live-filtered.txt  
```  
  
Extract just the URLs for downstream tools:  
```bash  
awk '{print $1}' live-filtered.txt > live-urls.txt  
```  
  
---  
  
## Step 8 — Worth Testing Assessment  
  
Run each check against `live-urls.txt`. Results from all checks feed the final prioritized list in Step 9.  
  
### A — Subdomain Takeover Check  
Detects dangling CNAMEs pointing to unclaimed cloud services. Fast and high-impact — run first:  
```bash  
subzy run --targets live-urls.txt --hide_fails --concurrency 50 --https  
```  
  
Cross-check with Nuclei takeover templates:  
```bash  
nuclei -l live-urls.txt --t ~/nuclei-templates/http/takeovers/ -o takeover-results.txt  
```  
Any hit here is a potential valid finding — note it immediately before continuing.  
  
### B — Tech Stack Detection  
Identifies frameworks, CMS versions, and libraries. Outdated or known-vulnerable stacks jump the queue:  
```bash  
httpx -l live-urls.txt \  
 -silent \  
 -tech-detect \  
 -status-code \  
 -title \  
 -o tech-detected.txt  
```  
  
Flag these tech patterns for priority testing:  
```bash  
grep -iE "wordpress|drupal|joomla|laravel|spring|struts|jenkins|grafana|kibana|jira|confluence|tomcat" \  
 tech-detected.txt > priority-tech.txt  
```  
  
### C — Screenshots  
Visual triage — faster than visiting each subdomain manually. Spot admin panels, login pages, and default installs at a glance:  
```bash  
gowitness scan file -f live-urls.txt \
  --timeout 10 \
  --threads 20 \
  --screenshot-path screenshots/ \
  --write-db
  
# Open the report in browser  
gowitness report server --db-uri sqlite://gowitness.sqlite3 
```  
  
### D — WAF and CDN Detection  
Know what's in front of each target before testing. CDN-fronted subdomains have different constraints; WAF-free targets are highe  
r priority:  
```bash  
# CDN/WAF detection via httpx (per subdomain)  
httpx -list passive-resolved-clean.txt -cdn -o cdn-waf-detected.txt
  
# Separate WAF-free from CDN-fronted  
grep -v "cloudflare\|fastly\|akamai\|cloudfront\|incapsula" cdn-waf-detected.txt | cut -d ' ' -f1 > no-cdn.txt  
```  
  
For specific WAF fingerprinting on high-value targets:  
```bash  
wafw00f https://target-subdomain.com  
```  
  
### F — Security Header Analysis

Collect headers with curl — one line per URL for easy grepping. Checks cover transport security, information disclosure, CORS, cache behaviour, and cookie flags.

#### Collect headers
```bash
while IFS= read -r url; do
    headers=$(curl -sI --max-time 10 -A "Mozilla/5.0" "$url" 2>/dev/null \
        | tr -d '\r' | tr '[:upper:]' '[:lower:]' | tr '\n' ' ')
    echo "$url $headers"
done < live-urls.txt > headers-flat.txt
```

#### Missing defensive headers
Each command outputs only the URLs where that header is absent:
```bash
# HSTS missing — HTTP downgrade and MitM attacks possible
awk '!/strict-transport-security/{print $1}' headers-flat.txt > missing-hsts.txt

# CSP missing — no injection policy, XSS easier
awk '!/content-security-policy/{print $1}' headers-flat.txt > missing-csp.txt

# X-Frame-Options missing and no CSP frame-ancestors — clickjacking possible
awk '!/x-frame-options/ && !/frame-ancestors/{print $1}' headers-flat.txt > missing-xfo.txt

# X-Content-Type-Options missing — MIME sniffing attacks possible
awk '!/x-content-type-options/{print $1}' headers-flat.txt > missing-xcto.txt

# Permissions-Policy missing — browser features unrestricted
awk '!/permissions-policy/{print $1}' headers-flat.txt > missing-permspol.txt
```

#### Weak header values
Present but misconfigured — still exploitable:
```bash
# CSP weakened by unsafe directives — XSS mitigations bypassed
grep "content-security-policy" headers-flat.txt \
    | grep -E "unsafe-inline|unsafe-eval" \
    | awk '{print $1}' > weak-csp.txt

# HSTS max-age under 1 year — short window allows downgrade attacks
grep "strict-transport-security" headers-flat.txt \
    | grep -Eo "max-age=[0-9]+" \
    | awk -F= '$2+0 < 31536000' > weak-hsts-maxage.txt
```

#### Information disclosure
Headers that reveal tech stack — aids fingerprinting and CVE lookup:
```bash
# Server header — reveals software and version (Apache/2.4.x, nginx/1.x, etc.)
grep " server:" headers-flat.txt | awk '{print $1}' > info-server.txt

# X-Powered-By — PHP version, Express, ASP.NET, etc.
grep "x-powered-by:" headers-flat.txt | awk '{print $1}' > info-powered-by.txt

# ASP.NET version disclosure
grep -E "x-aspnet-version:|x-aspnetmvc-version:" headers-flat.txt | awk '{print $1}' > info-aspnet.txt

# X-Runtime — Rails response time (timing oracle for blind injection attacks)
grep "x-runtime:" headers-flat.txt | awk '{print $1}' > info-runtime.txt

# X-Debug-Token / X-Debug-Token-Link — Symfony profiler exposed
grep "x-debug-token" headers-flat.txt | awk '{print $1}' > info-debug-token.txt

# X-Generator — CMS or framework fingerprint
grep "x-generator:" headers-flat.txt | awk '{print $1}' > info-generator.txt
```

Any version number in `info-server.txt` or `info-powered-by.txt` is worth CVE-checking immediately.

#### CORS headers
```bash
# Wildcard origin — any domain can read responses
grep "access-control-allow-origin: \*" headers-flat.txt | awk '{print $1}' > cors-wildcard.txt

# Credentials flag true — combined with a permissive origin this is critical
grep "access-control-allow-credentials: true" headers-flat.txt | awk '{print $1}' > cors-credentials.txt
```

Any URL in both `cors-wildcard.txt` and `cors-credentials.txt` is a high-priority CORS finding.

#### Cache headers
```bash
# Cache-Control missing — potential cache poisoning surface
awk '!/cache-control/{print $1}' headers-flat.txt > missing-cache-control.txt

# X-Cache present — confirms a caching layer exists (Varnish, Nginx, CDN)
grep " x-cache:" headers-flat.txt | awk '{print $1}' > has-cache-layer.txt

# Age header present — response was served from cache
grep " age:" headers-flat.txt | awk '{print $1}' > served-from-cache.txt
```

Targets in both `has-cache-layer.txt` and `missing-cache-control.txt` are the highest-priority cache poisoning candidates.

#### Cookie flags
```bash
# Set-Cookie without HttpOnly — cookie readable via JavaScript (XSS session theft)
grep "set-cookie:" headers-flat.txt \
    | grep -v "httponly" \
    | awk '{print $1}' > cookie-no-httponly.txt

# Set-Cookie without Secure — session cookie transmitted over plain HTTP
grep "set-cookie:" headers-flat.txt \
    | grep -v "; secure" \
    | awk '{print $1}' > cookie-no-secure.txt

# Set-Cookie without SameSite — CSRF and cross-site request leakage possible
grep "set-cookie:" headers-flat.txt \
    | grep -v "samesite" \
    | awk '{print $1}' > cookie-no-samesite.txt
```

#### Combine all findings for tier building
```bash
cat missing-hsts.txt missing-csp.txt missing-xfo.txt missing-xcto.txt \
    weak-csp.txt \
    info-server.txt info-powered-by.txt info-aspnet.txt info-debug-token.txt \
    cors-wildcard.txt cors-credentials.txt \
    missing-cache-control.txt \
    cookie-no-httponly.txt cookie-no-secure.txt cookie-no-samesite.txt \
    | sort -u > missing-headers.txt
```
  
---  
  
## Step 9 — Final Prioritized List  
  
Build a tiered working list from all Step 8 findings. Test in tier order.  
  
```bash  
# Tier 1 — Immediate  
# Takeover candidates + auth-gated + WAF-free + missing headers  
cat takeover-results.txt \  
 <(grep " 401\| 403" live-filtered.txt) \  
 no-cdn.txt \  
 missing-csp.txt \  
 | awk '{print $1}' | sort -u > tier1.txt  
  
# Tier 2 — High value  
# Interesting tech stack + keyword patterns in subdomain name  
cat priority-tech.txt \  
 <(grep -iE "\.(admin|login|dev|staging|api|internal|portal|dashboard|uat|test)\." live-filtered.txt) \  
 | awk '{print $1}' | sort -u > tier2.txt  
  
# Tier 3 — Standard  
# Everything live that didn't make Tier 1 or 2  
comm -23 <(awk '{print $1}' live-filtered.txt | sort) \  
 <(cat tier1.txt tier2.txt | sort) > tier3.txt  
```  
  
Final counts:  
```bash  
echo "Tier 1: $(wc -l < tier1.txt)"  
echo "Tier 2: $(wc -l < tier2.txt)"  
echo "Tier 3: $(wc -l < tier3.txt)"  
```  
  
| Tier | Criteria | Test When |  
|---|---|---|  
| 1 | Takeover candidate, auth-gated (401/403), WAF-free, missing security headers | Immediately |  
| 2 | Interesting tech stack, admin/dev/api/staging/portal in name | After Tier 1 |  
| 3 | Standard live subdomains, CDN-fronted | After Tier 2 |  
  
---  
  
## Notes  
  
<!-- Add per-domain findings, wildcard status, interesting subdomains, and takeover candidates here -->