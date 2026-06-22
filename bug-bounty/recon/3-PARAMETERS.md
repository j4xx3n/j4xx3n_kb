# Parameter Discovery Checklist  
  
> **Goal:** Collect every URL parameter the target exposes — from archives, live crawling, JavaScript files, and active fuzzing. Nothing is filtered out. The final output is a complete, unmodified URL list that feeds directly into attack-specific testing. Input is the live subdomain lists (`tier1.txt`, `tier2.txt`, `tier3.txt`) from the subdomain phase.  
  
---  
  
## Step 1 — Pre-Flight  
  
- [ ] Set up a working directory for this target:  
 ```bash  
 mkdir -p params/{passive,crawl,js,fuzzing}  
 cd params/  
 ```  
- [ ] Confirm the live subdomain lists exist from the previous phase:  
 ```bash  
 wc -l ~/recon/target/tier1.txt ~/recon/target/tier2.txt ~/recon/target/tier3.txt  
 ```  
- [ ] Build a combined scope file from all three tiers:  
 ```bash  
 cat ~/recon/target/tier1.txt ~/recon/target/tier2.txt ~/recon/target/tier3.txt \  
   | sort -u > scope.txt  
 ```  
  
---  
  
## Step 2 — Passive Archive Harvesting  
  
No direct contact with the target. Pull every URL these services have ever indexed. Run all tools — each catches sources the others miss.  
  
### waybackurls  
Fetches all URLs the Wayback Machine knows about:  
```bash  
cat scope.txt | waybackurls > passive/wayback.txt  
wc -l passive/wayback.txt  
```  
  
### gau  
Aggregates from Wayback Machine, Common Crawl, AlienVault OTX, and URLScan:  
```bash  
cat scope.txt | gau --threads 10 --subs > passive/gau.txt  
wc -l passive/gau.txt  
```  
  
### waymore  
The most thorough archive tool — also downloads actual response bodies from the archive, which often contain developer comments, hidden pa  
rams, and deprecated endpoints:  
```bash  
# URL mode — fast pass first  
waymore -i target.com -mode U -oU passive/waymore-urls.txt  
  
# Response download mode — run after URL pass, limit to avoid massive downloads  
waymore -i target.com -mode R -l 1000 -oR passive/waymore-responses/  
```  
  
Extract additional URLs and params from downloaded response bodies:  
```bash  
# Run LinkFinder over archived responses (see Step 4 for full JS analysis)  
xnLinkFinder -i passive/waymore-responses/ -sp https://target.com \  
 -o passive/waymore-endpoints.txt  
```  
  
### ParamSpider  
Mines parameter names specifically from Wayback CDX data — good at surfacing param names that gau/waybackurls normalize away:  
```bash  
paramspider -d target.com -o passive/paramspider.txt
```  
  
---  
  
## Step 3 — Live Site Crawling  
  
Spider the target as it exists right now. Archives miss recently deployed endpoints; crawling catches them.  
  
### Katana (primary crawler)  
Modern, fast, JavaScript-aware crawler from ProjectDiscovery:  
```bash  
# Standard crawl with JS parsing enabled  
katana -list scope.txt -d 3 -jc -silent -o crawl/katana.txt  
  
# Increase depth for larger targets  
katana -list scope.txt -d 5 -jc -kf all -silent -o crawl/katana-deep.txt  

# Advanced crawl
katana -silent -rl 20 -list scope.txt -d 7 -jc -jsl -kf all -retry 2 -pc -ef woff,css,png,svg,jpg,woff2,jpeg,gif -fx -nc 2>&1 | anew crawl/katana.txt
```

  
### Gospider  
Good at following links katana misses — different parsing logic:  
```bash  
gospider -S scope.txt -d 3 -c 10 -o crawl/gospider/ --no-redirect  
cat crawl/gospider/*.txt 2>/dev/null > crawl/gospider-merged.txt  
```  
  
### Hakrawler  
Lightweight — fast pass for any scope.txt target:  
```bash  
cat scope.txt | hakrawler -d 3 -subs > crawl/hakrawler.txt  
```  
  
### Burp Suite — Proxy Crawl  
The crawler above are automated. Burp's crawl captures what only a human browser session reveals — authenticated pages, form submissions,  
SPA state transitions:  
- [ ] Set browser proxy to `127.0.0.1:8080`  
- [ ] Browse the target manually — log in, trigger key workflows, visit every page  
- [ ] In Burp: **Target → Site map** — right-click the target host → **Crawl**  
- [ ] After crawl completes: **Target → Site map** → right-click → **Copy links** → save to `crawl/burp-sitemap.txt`  
- [ ] Extract all URLs with params from Burp: **Target → Site map** → filter by "Items with parameters" → export  
  
---  
  
## Step 4 — JavaScript File Analysis  
  
JavaScript files often contain undocumented API endpoints and parameter names that never appear in any archive or crawl.  
  
### Collect all JS file URLs  
```bash  
cat crawl/katana.txt crawl/gospider-merged.txt crawl/hakrawler.txt passive/gau.txt | grep -Ei "\.js(\?|$)" | sort -u | httpx-toolkit -mc 200 > js/js-urls.txt  
  
wc -l js/js-urls.txt  
```  
  
### LinkFinder — extract endpoints and params  
```bash  
# Run against entire domain (covers JS files linked from all pages)  
python3 linkfinder.py -i https://target.com -d -o cli > js/linkfinder-domain.txt  
  
# Run against each collected JS URL individually  
while IFS= read -r jsurl; do  
 python3 linkfinder.py -i "$jsurl" -o cli 2>/dev/null  
done < js/js-urls.txt | sort -u > js/linkfinder-files.txt  
  
# Combine  
cat js/linkfinder-domain.txt js/linkfinder-files.txt | sort -u > js/endpoints.txt  
```  
  
### SecretFinder — hunt for hardcoded secrets in JS  
Not for parameter discovery but run it now while the JS files are collected — finds API keys, tokens, and credentials hardcoded in source:  
```bash  
while IFS= read -r jsurl; do  
 python3 SecretFinder.py -i "$jsurl" -o cli 2>/dev/null  
done < js/js-urls.txt | sort -u > js/secrets.txt  
  
wc -l js/secrets.txt  
# Manually review js/secrets.txt — any real credentials are immediate findings  
```  
  
---  
  
## Step 5 — Consolidate and Clean  
  
Merge everything, deduplicate, validate what is currently live.  
  
### Merge all sources  
```bash  
cat passive/wayback.txt \  
   passive/gau.txt \  
   passive/waymore-urls.txt \  
   passive/waymore-endpoints.txt \  
   passive/paramspider.txt \  
   crawl/katana.txt \  
   crawl/gospider-merged.txt \  
   crawl/hakrawler.txt \  
   crawl/burp-sitemap.txt \  
   js/endpoints.txt \  
 | sort -u > all-urls-raw.txt  
  
wc -l all-urls-raw.txt  
```  
  
### Split into parameterized and clean endpoints  
```bash  
# URLs that already have at least one parameter  
grep "?" all-urls-raw.txt | sort -u > urls-with-params.txt  
  
# Clean endpoints — no params yet (targets for Step 7 fuzzing)  
grep -v "?" all-urls-raw.txt | grep -Ei "^https?://" | sort -u > urls-no-params.txt  
  
wc -l urls-with-params.txt  
wc -l urls-no-params.txt  
```  
  
### Filter to confirmed scope  
```bash  
# Replace "target\.com" with the actual domain pattern  
grep -iE "target\.com" urls-with-params.txt | sort -u > params-in-scope.txt  
grep -iE "target\.com" urls-no-params.txt   | sort -u > endpoints-in-scope.txt  
```  
  
### Validate which parameterized URLs are currently live  
```bash  
httpx -l params-in-scope.txt -silent -mc 200,301,302,401,403 -o live-params.txt  

wc -l live-params.txt  
```  
  
---  
  
## Step 6 — Custom Wordlist Generation  
  
Build a target-specific parameter wordlist from everything collected so far. This wordlist will contain parameter names unique to this tar  
get's codebase — things no generic wordlist has.  
  
```bash  
# Extract every parameter name from all collected URLs  
cat urls-with-params.txt | unfurl format %q | tr '&' '\n' | cut -d '=' -f1 | grep -Ev "^$|^[0-9]+$" | awk 'length < 40' | sort -u > fuzzing/custom-params.txt

wc -l fuzzing/custom-params.txt  
```  
  
Merge with standard wordlists for full coverage:  
```bash  
cat fuzzing/custom-params.txt \  
   ~/wordlists/arjun-params.txt \  
   ~/wordlists/params-top1m-assetnote.txt \  
 | sort -u > fuzzing/merged-params.txt  
  
wc -l fuzzing/merged-params.txt  
```  
  
---  
  
## Step 7 — Active Parameter Fuzzing  
  
Before running parameter discovery tools, expand the endpoint surface and sort what is worth fuzzing. Arjun and x8 test thousands of parameters per request — running them against the wrong endpoints wastes time and burns rate limits. Steps 7a–7e build the tiered target lists that every tool below consumes.

---

### 7a — robots.txt & sitemap.xml Harvest

These files expose paths that never appear in archives or crawls.

```bash
mkdir -p fuzzing/endpoints

# Parse Allow/Disallow entries from robots.txt across all hosts
xargs -a scope.txt -P 50 -I {} sh -c '    
 curl -s "{}/robots.txt" 2>/dev/null \  
   | grep -Ei "^(Allow|Disallow):" \  
   | awk "{print \$2}" \  
   | grep -v "^\*$" \  
   | sed "s|^|{}|"  
' >> fuzzing/endpoints/robots-paths.txt

# Extract all <loc> URLs from sitemap.xml
xargs -a scope.txt -P 50 -I {} sh -c '
  curl -s "{}/sitemap.xml" 2>/dev/null \
    | grep -oE "https?://[^<\"]+"
' >> fuzzing/endpoints/sitemap-urls.txt

sort -u fuzzing/endpoints/robots-paths.txt \
        fuzzing/endpoints/sitemap-urls.txt \
     > fuzzing/endpoints/robots-sitemap.txt

wc -l fuzzing/endpoints/robots-sitemap.txt
```

---

### 7b — Active Path Fuzzing with FFUF

Archives and crawls only surface paths that existed historically or that the crawler followed. FFUF actively brute-forces routes that were never indexed: admin panels, backup files, deprecated endpoints.

- **`raft-medium-words.txt`** (~63k words) — default for all in-scope hosts. Fast enough to run across every subdomain in `scope.txt`.
- **`raft-large-words.txt`** (~119k words) — for high-value single targets where thoroughness matters more than speed. Run against `tier1.txt` hosts only.

```bash
# Medium pass — run against every in-scope host
while IFS= read -r host; do
  domain=$(echo "$host" | sed 's|https\?://||;s|/.*||')
  ffuf -w ~/wordlists/raft-medium-words.txt \
    -u "${host}/FUZZ" \
    -mc 200,201,204,301,302,307,401,403,405 \
    -fc 404 \
    -ac \
    -t 30 \
    -o "fuzzing/endpoints/ffuf-medium-${domain}.json" -of json 2>/dev/null
done < scope.txt

# Large pass — run against tier-1 subdomains only (highest-value hosts from subdomain phase)
while IFS= read -r host; do
  domain=$(echo "$host" | sed 's|https\?://||;s|/.*||')
  ffuf -w ~/wordlists/raft-large-words.txt \
    -u "${host}/FUZZ" \
    -mc 200,201,204,301,302,307,401,403,405 \
    -fc 404 \
    -ac \
    -t 30 \
    -o "fuzzing/endpoints/ffuf-large-${domain}.json" -of json 2>/dev/null
done < ~/recon/target/tier1.txt

# Extract all discovered paths from every FFUF output file
jq -r '.results[].url' fuzzing/endpoints/ffuf-*.json 2>/dev/null \
  | sort -u > fuzzing/endpoints/ffuf-discovered.txt

wc -l fuzzing/endpoints/ffuf-discovered.txt
```

---

### 7c — Merge and Validate

```bash
cat endpoints-in-scope.txt \
    fuzzing/endpoints/ffuf-discovered.txt \
    fuzzing/endpoints/robots-sitemap.txt \
  | grep -Ei "^https?://" \
  | sort -u > fuzzing/endpoints/all-endpoints.txt

wc -l fuzzing/endpoints/all-endpoints.txt

# Confirm what is currently live
httpx -l fuzzing/endpoints/all-endpoints.txt \
  -silent -mc 200,301,302,401,403 \
  -o fuzzing/endpoints/live-all.txt

wc -l fuzzing/endpoints/live-all.txt
```

---

### 7d — Flag Empty-200 Endpoints

A `200 OK` with a very small response body often means the endpoint is waiting for a parameter before it does anything. These are high-value fuzzing targets regardless of what the path name looks like — and one of the most commonly overlooked signals.

```bash
# Get content-length for all live 200 responses
httpx -l fuzzing/endpoints/live-all.txt \
  -silent -mc 200 \
  -cl \
  -o fuzzing/endpoints/live-200-sizes.txt

# Isolate responses under 500 bytes
awk '$NF < 500 {print $1}' fuzzing/endpoints/live-200-sizes.txt \
  | sort -u > fuzzing/endpoints/empty-200.txt

wc -l fuzzing/endpoints/empty-200.txt
# Manually glance at this list — false positives include redirect stubs and blank templates
```

---

### 7e — Build Priority Tiers

Split `live-all.txt` into three tiers. Arjun and x8 run only against Tier 1. FFUF parameter fuzzing covers Tier 2. Tier 3 is skipped unless something looks unusual on manual review.

**Tier 1 — Arjun + x8 + FFUF (highest value)**
Authentication, search, user data, admin, forms — anything the app actually processes input for:
```bash
grep -iE \
  "/login|/signin|/auth|/oauth|/sso|/saml|\
/register|/signup|\
/account|/profile|/user|/member|/customer|\
/search|/query|/filter|/find|/results|\
/admin|/manage|/dashboard|/panel|/control|/backend|\
/upload|/import|/export|\
/reset|/forgot|/password|/recover|\
/settings|/config|/prefs|/preferences|\
/order|/cart|/checkout|/payment|/invoice|\
/contact|/submit|/send|/feedback|\
/api/|/v[0-9]+/" \
  fuzzing/endpoints/live-all.txt | sort -u > fuzzing/endpoints/tier1.txt

# Merge in all empty-200 endpoints — these go to Tier 1 regardless of path name
cat fuzzing/endpoints/tier1.txt fuzzing/endpoints/empty-200.txt \
  | sort -u > fuzzing/endpoints/tier1.txt.tmp \
  && mv fuzzing/endpoints/tier1.txt.tmp fuzzing/endpoints/tier1.txt

wc -l fuzzing/endpoints/tier1.txt
```

**Tier 2 — FFUF only (medium value)**
Dynamic-looking paths not in Tier 1 — content pages, slugs, numeric segments (IDOR surface):
```bash
grep -vxFf fuzzing/endpoints/tier1.txt fuzzing/endpoints/live-all.txt \
  | grep -Ei \
    "/page|/view|/show|/detail|/item|/post|/article|\
/category|/product|/news|/blog|/topic|\
/[0-9]+(/|$)|/[a-z0-9_-]+-[0-9]+(/|$)" \
  | sort -u > fuzzing/endpoints/tier2.txt

wc -l fuzzing/endpoints/tier2.txt
```

**Tier 3 — Skip**
Everything remaining. Glance at this list before discarding — move anything that looks dynamic up to Tier 2:
```bash
grep -vxFf fuzzing/endpoints/tier1.txt fuzzing/endpoints/live-all.txt \
  | grep -vxFf fuzzing/endpoints/tier2.txt \
  | grep -vEi "\.(css|js|png|jpg|jpeg|gif|svg|ico|woff|woff2|ttf|pdf|xml|txt)$" \
  | sort -u > fuzzing/endpoints/tier3-skipped.txt

wc -l fuzzing/endpoints/tier3-skipped.txt
```

**Summary:**
```bash
echo "Tier 1 (Arjun + x8 + FFUF): $(wc -l < fuzzing/endpoints/tier1.txt)"
echo "Tier 2 (FFUF only):          $(wc -l < fuzzing/endpoints/tier2.txt)"
echo "Tier 3 (skipped):            $(wc -l < fuzzing/endpoints/tier3-skipped.txt)"
```

---

### Arjun — hidden GET / POST / JSON parameter discovery  
Arjun tests up to 25,890 parameter names in ~50 requests by grouping them and using response diffing to identify which ones the server act  
ually processes:  
```bash  
# Single GET endpoint  
arjun -u https://target.com/endpoint -m GET -w fuzzing/merged-params.txt  
  
# Single POST endpoint  
arjun -u https://target.com/endpoint -m POST -w fuzzing/merged-params.txt  
  
# JSON API endpoint  
arjun -u https://target.com/api/v1/resource -m JSON  
  
# XML endpoint  
arjun -u https://target.com/api/resource -m XML  
  
# Bulk from file — run against Tier 1 endpoints only  
arjun -i fuzzing/endpoints/tier1.txt -w fuzzing/merged-params.txt -oJ fuzzing/arjun-results.json  
  
# Stealth mode for sensitive targets — slow but avoids triggering WAF/rate limits  
arjun -u https://target.com/endpoint -d 3 --rate-limit 3 --stable  
```  
  
### x8 — faster Rust-based alternative  
Use when Arjun is too slow or when response sizes make diffing tricky:  
```bash  
# GET  
x8 -u "https://target.com/endpoint" -w fuzzing/merged-params.txt -o fuzzing/x8-get.txt  
  
# POST form-encoded  
x8 -u "https://target.com/endpoint" -X POST \  
  -H "Content-Type: application/x-www-form-urlencoded" \  
  -w fuzzing/merged-params.txt -o fuzzing/x8-post.txt  
  
# POST JSON body  
x8 -u "https://target.com/api/resource" -X POST \  
  --data '{}' \  
  -H "Content-Type: application/json" \  
  -w fuzzing/merged-params.txt -o fuzzing/x8-json.txt  
```  
  
### FFUF — parameter name fuzzing with response-size baseline  
FFUF is faster than Arjun/x8 for large wordlists but needs a baseline response size to filter noise:  
  
```bash  
# Step 1: establish baseline size for the endpoint  
curl -s -o /dev/null -w "%{size_download}" "https://target.com/endpoint"  
  
# Step 2: GET parameter name discovery — filter out responses matching baseline size  
ffuf -w fuzzing/merged-params.txt \  
 -u "https://target.com/endpoint?FUZZ=test" \  
 -mc 200,301,302,401,403 \  
 -fs <baseline_size> \  
 -t 25 \  
 -o fuzzing/ffuf-get.json -of json  
  
# POST parameter name discovery  
ffuf -w fuzzing/merged-params.txt \  
 -u "https://target.com/endpoint" \  
 -X POST \  
 -d "FUZZ=test" \  
 -H "Content-Type: application/x-www-form-urlencoded" \  
 -mc 200,301,302,401,403 \  
 -fs <baseline_size> \  
 -t 25 \  
 -o fuzzing/ffuf-post.json -of json  
  
# POST JSON body parameter discovery  
ffuf -w fuzzing/merged-params.txt \  
 -u "https://target.com/api/resource" \  
 -X POST \  
 -d '{"FUZZ":"test"}' \  
 -H "Content-Type: application/json" \  
 -mc 200,301,302,401,403 \  
 -fs <baseline_size> \  
 -t 25 \  
 -o fuzzing/ffuf-json.json -of json  
  
# Rate-limit bypass for WAF-protected endpoints  
ffuf -w fuzzing/merged-params.txt \  
 -u "https://target.com/endpoint?FUZZ=test" \  
 -mc 200,301,302,401,403 \  
 -fs <baseline_size> \  
 -p 0.5-1.5 -t 10  
```  

### FFUF — Tier 2 batch (medium-value endpoints)

Run GET parameter fuzzing across every Tier 2 endpoint. `-ac` auto-calibrates the baseline per URL so no manual size measurement is needed:
```bash
while IFS= read -r url; do
  ffuf -w fuzzing/merged-params.txt \\
    -u "${url}?FUZZ=test" \\
    -mc 200,301,302,401,403 \\
    -ac \\
    -t 20 \\
    -o "fuzzing/ffuf-tier2-$(echo "$url" | md5sum | cut -c1-8).json" -of json 2>/dev/null
done < fuzzing/endpoints/tier2.txt

# Merge all Tier 2 results into a single file
jq -r '.results[].url' fuzzing/ffuf-tier2-*.json 2>/dev/null \\
  | sort -u > fuzzing/ffuf-tier2-all.txt
```

### Burp Suite — Param Miner (unkeyed and hidden parameters)  
Param Miner uses binary search + response diffing to find up to 65,536 parameters per request. It is the best tool for finding unkeyed par  
ameters — parameters the server processes but does not include in the cache key (critical for web cache poisoning):  
- [ ] Send an interesting request to Burp Repeater  
- [ ] Right-click the request → **Extensions → Param Miner → Guess (GET/POST) parameters**  
- [ ] Also run: **Guess headers** — finds unkeyed headers and routing headers (`X-Forwarded-Host`, `X-Original-URL`, etc.)  
- [ ] Monitor the **Extensions → Param Miner** output tab for results  
- [ ] For bulk scanning: **Target → Site map** → select multiple requests → right-click → **Guess parameters** — runs across the entire si  
te map  
- [ ] Add discovered parameters to `fuzzing/paramminer-results.txt`  
  
---  
  
## Step 8 — Final Output  
  
Merge everything into one complete, unfiltered parameter list.  
  
```bash  
# Combine live archived params with fuzzing discoveries  
cat live-params.txt \  
   fuzzing/arjun-results.json \  
   fuzzing/x8-get.txt \  
   fuzzing/x8-post.txt \  
   fuzzing/ffuf-get.json \  
   fuzzing/paramminer-results.txt \  
 | grep -Ei "^https?://" \  
 | sort -u > all-params-final.txt  
  
wc -l all-params-final.txt  
```  
  
The full `all-params-final.txt` is the primary output — nothing is removed from it. The grep commands below create **supplementary priorit  
y review files only**. They do not modify the main list.  
  
```bash  
# Create review files for manual prioritization — additive, not filters  
grep -iE "id=|user_?id=|account=|order=|invoice=|doc=" all-params-final.txt  > review/idor.txt  
grep -iE "redirect=|return=|next=|url=|dest=|continue=|out=" all-params-final.txt > review/redirect.txt  
grep -iE "file=|path=|include=|page=|template=|dir="      all-params-final.txt > review/lfi.txt  
grep -iE "callback=|webhook=|server=|uri=|api=|endpoint="  all-params-final.txt > review/ssrf.txt  
grep -iE "debug=|admin=|test=|enable=|disable=|internal="  all-params-final.txt > review/debug.txt  
grep -iE "login|auth|oauth|token|session|sso|saml|state="  all-params-final.txt > review/auth.txt  
```  
  
Final counts:  
```bash  
echo "Total parameters: $(wc -l < all-params-final.txt)"  
for f in review/*.txt; do echo "$(wc -l < $f) - $f"; done | sort -rn  
```  
  
**The full `all-params-final.txt` list feeds into every attack-specific checklist that follows.**  
  
---  
  
## Notes  
  
<!-- Add notable discovered parameters, unusual endpoints, JS secrets, and Param Miner findings here -->