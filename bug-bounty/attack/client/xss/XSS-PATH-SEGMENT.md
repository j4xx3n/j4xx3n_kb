# Path Segment Reflection — Recon Checklist

**Goal:** From millions of Wayback Machine URLs, identify URL path segments that are
(a) likely attacker-controllable and (b) reflected in the live HTTP response (body or headers).

**Not in scope:** Payload injection, exploit verification, WAF bypass. This is pure recon.

**Speed vs precision:** Optimized for coverage. False positives are acceptable; false negatives are not.

---

## Definitions

| Term                              | Meaning here                                                                                                                                                      |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Attacker-controllable segment** | A path position whose value varies across real requests (captured in Wayback) and is not a static route keyword — i.e., the app uses whatever the user puts there |
| **Reflected**                     | The segment's literal value (URL-decoded) appears verbatim in the live response body or in a response header value                                                |
| **Pattern**                       | A URL normalized to a template, e.g. `/blog/{int}/{slug}`, by replacing variable positions with placeholders                                                      |
| **Cluster**                       | The group of all Wayback URLs that share the same pattern                                                                                                         |

---

## Required Tools

Install before starting. All available on Kali or via `go install`.

- [ ] `gau` — pull URLs from Wayback + Common Crawl
- [ ] `uro` — URL deduplication / normalization (pip install uro)
- [ ] `httpx` (projectdiscovery) — live probing, header capture, body match
- [ ] `unfurl` — URL component parsing / path extraction
- [ ] `anew` — append unique lines (dedup streams)
- [ ] `python3` + `aiohttp` — async reflection-check script (Stage 6)
- [ ] `jq` — JSON manipulation for httpx output
- [ ] Burp Suite Pro — final manual triage

Verify installs:
```bash
gau --version && uro --version && httpx -version && unfurl -version
```

---

## Stage 0 — Scope Setup

Define your target before touching any tool.

- [ ] Set the target domain (or domain list):
  ```bash
  export TARGET="example.com"          # single domain
  export SCOPE_FILE="domains.txt"      # or a file of domains
  ```
- [ ] Create working directory structure:
  ```bash
  mkdir -p recon/{raw,processed,live,reflected,burp}
  ```
- [ ] If authenticated testing is in scope, capture session credentials now (Stage 5B):
  - [ ] Log in via browser → copy `Cookie:` header value from Burp → save to `recon/auth_cookie.txt`
  - [ ] Or: copy `Authorization: Bearer <token>` → save to `recon/auth_header.txt`

---

## Stage 1 — URL Collection

Pull historical URLs from Wayback Machine and Common Crawl.

- [ ] Collect with `gau` (covers Wayback + Common Crawl + OTX):
  ```bash
  gau --threads 10 --subs "$TARGET" \
      --blacklist ttf,woff,woff2,svg,png,jpg,jpeg,gif,ico,css,pdf,zip,gz \
      --o recon/raw/gau_raw.txt \
      "$TARGET"
  ```
  For a domain list:
  ```bash
  cat "$SCOPE_FILE" | gau --threads 10 \
      --blacklist ttf,woff,woff2,svg,png,jpg,jpeg,gif,ico,css,pdf,zip,gz \
      --o recon/raw/gau_raw.txt
  ```

- [ ] Optional: also pull with `waybackurls` and merge (catches gaps):
  ```bash
  echo "$TARGET" | waybackurls >> recon/raw/gau_raw.txt
  ```

- [ ] Check raw count:
  ```bash
  wc -l recon/raw/gau_raw.txt
  ```

---

## Stage 2 — Preprocessing & Normalization

Reduce millions of URLs to a manageable working set before any HTTP requests.

### 2A — Basic Filtering

- [ ] Remove obviously static URLs by extension:
  ```bash
  grep -viE '\.(css|js|mjs|map|png|jpg|jpeg|gif|svg|ico|woff|woff2|ttf|eot|pdf|zip|gz|tar|mp4|mp3|webp|avif)(\?|$|#)' \
      recon/raw/gau_raw.txt > recon/processed/no_static.txt
  ```

- [ ] Keep only http/https:
  ```bash
  grep -E '^https?://' recon/processed/no_static.txt > recon/processed/http_only.txt
  ```

- [ ] Remove URLs with no path beyond root (nothing to reflect):
  ```bash
cat > filter_paths.py << 'EOF'
import sys
from urllib.parse import urlparse

with open("recon/processed/http_only.txt") as f:
    for line in f:
        url = line.strip()
        try:
            p = urlparse(url)
            segments = [s for s in p.path.split('/') if s]
            if len(segments) >= 1:
                print(url)
        except Exception:
            pass
EOF

python3 filter_paths.py > recon/processed/has_path.txt
  ```

### 2B — URL Deduplication with uro

`uro` removes URLs that are structurally redundant (same path pattern, different param values).

- [ ] Run uro:
  ```bash
  uro -i recon/processed/has_path.txt -o recon/processed/uro_deduped.txt
  ```

- [ ] Check reduction:
  ```bash
  echo "Before: $(wc -l < recon/processed/has_path.txt)"
  echo "After:  $(wc -l < recon/processed/uro_deduped.txt)"
  ```

### 2C — Path Segment Extraction

Extract the path-only portion and parse segments per URL.

- [ ] Extract path per URL with unfurl:
  ```bash
  unfurl --unique paths recon/processed/uro_deduped.txt \
      > recon/processed/paths_only.txt
  ```

- [ ] Keep the full URL alongside its parsed path for later:
  ```bash
  python3 - <<'EOF'
  from urllib.parse import urlparse, unquote
  import json

  out = []
  with open("recon/processed/uro_deduped.txt") as f:
      for line in f:
          url = line.strip()
          try:
              p = urlparse(url)
              raw_segs = [s for s in p.path.split('/') if s]
              decoded_segs = [unquote(s) for s in raw_segs]
              out.append(json.dumps({
                  "url": url,
                  "raw_segments": raw_segs,
                  "decoded_segments": decoded_segs,
                  "seg_count": len(raw_segs)
              }))
          except Exception:
              pass

  with open("recon/processed/urls_with_segments.jsonl", "w") as f:
      f.write('\n'.join(out))
  EOF
  ```

---

## Stage 3 — Attacker-Controllability Scoring

Identify which path positions are likely user-supplied, not static route keywords.
No HTTP requests in this stage — pure offline analysis.

### 3A — Known Static Segment Blocklist

Create a blocklist of common static route keywords to skip:
```bash
cat > recon/processed/static_segments.txt <<'EOF'
api v1 v2 v3 v4 v5 beta alpha admin login logout register auth oauth
callback redirect search users user account accounts profile settings
dashboard home index about contact help faq terms privacy docs
static assets media images img uploads files download en us uk fr de
es pt ru zh ja ko ar tr nl pl sv da fi nb it
health status ping metrics rpc graphql rest soap wsdl feed rss atom
sitemap robots.txt .well-known acme-challenge wp-admin wp-content
wp-includes themes plugins wp-json xmlrpc.php
EOF
```

### 3B — Cluster by Pattern & Score Variable Positions

This script:
1. Groups URLs by their path pattern (replaces variable segments with `{var}`)
2. Identifies which positions actually vary across the cluster
3. Scores each segment for attacker-controllability
4. Outputs one representative URL per cluster per variable position

```bash
python3 - <<'EOF'
import json, re, sys
from collections import defaultdict
from urllib.parse import unquote

# Load static segment blocklist
with open("recon/processed/static_segments.txt") as f:
    STATIC = set(f.read().split())

# Heuristics for "looks like user input"
def is_variable(seg, decoded):
    seg_l = decoded.lower().strip()
    if seg_l in STATIC:
        return False
    if len(decoded) < 4:          # very short = likely static keyword
        return False
    if re.match(r'^\d+$', decoded):    # pure integer ID
        return True
    if re.match(r'^[0-9a-f-]{32,}$', decoded, re.I):  # UUID-like
        return True
    if '-' in decoded and len(decoded) > 6:  # slug
        return True
    if '_' in decoded and len(decoded) > 6:  # underscore slug
        return True
    if re.search(r'[A-Z]', decoded) and re.search(r'[a-z]', decoded):  # camelCase
        return True
    if len(decoded) >= 8:         # long enough to be meaningful user input
        return True
    return False

# Load URLs
clusters = defaultdict(list)
with open("recon/processed/urls_with_segments.jsonl") as f:
    for line in f:
        entry = json.loads(line.strip())
        url = entry["url"]
        decoded_segs = entry["decoded_segments"]

        # Build a pattern key: replace variable-looking segments with {var}
        pattern_parts = []
        for s in decoded_segs:
            if is_variable(s, s):
                pattern_parts.append("{var}")
            else:
                pattern_parts.append(s.lower())
        pattern = "/" + "/".join(pattern_parts)
        clusters[pattern].append(entry)

# For each cluster, find variable positions and emit representative URLs
output = []
for pattern, entries in clusters.items():
    parts = pattern.split("/")[1:]  # skip leading empty from split
    for pos_idx, part in enumerate(parts):
        if part != "{var}":
            continue
        # Confirm this position actually varies (more than 1 unique value)
        values_at_pos = set()
        for e in entries:
            if pos_idx < len(e["decoded_segments"]):
                values_at_pos.add(e["decoded_segments"][pos_idx])
        if len(values_at_pos) < 2:
            continue  # position doesn't actually vary — not attacker-controlled
        # Pick a representative URL: prefer longer segment values (more distinct)
        best = max(entries, key=lambda e:
            len(e["decoded_segments"][pos_idx]) if pos_idx < len(e["decoded_segments"]) else 0)
        seg_value = best["decoded_segments"][pos_idx] if pos_idx < len(best["decoded_segments"]) else ""
        if len(seg_value) < 5:
            continue  # too short, high false-positive rate in reflection check
        output.append(json.dumps({
            "url": best["url"],
            "pattern": pattern,
            "variable_position": pos_idx,
            "segment_value": seg_value,
            "cluster_size": len(entries),
            "unique_values": len(values_at_pos)
        }))

with open("recon/processed/candidates.jsonl", "w") as f:
    f.write('\n'.join(output))

print(f"Candidate URLs with variable segments: {len(output)}")
EOF
```

- [ ] Extract just the URLs for live probing:
  ```bash
  jq -r '.url' recon/processed/candidates.jsonl | sort -u > recon/processed/candidate_urls.txt
  echo "Unique candidate URLs: $(wc -l < recon/processed/candidate_urls.txt)"
  ```

---

## Stage 4 — Live Host Filtering

Check which candidate URLs still return a live response.
No reflection logic yet — this step only filters dead URLs to save time in Stage 6.

### 4A — Unauthenticated Probe

- [ ] Run httpx with status code, content-type, and header capture:
  ```bash
  httpx -l recon/processed/candidate_urls.txt \
      -threads 100 \
      -rate-limit 150 \
      -timeout 10 \
      -retries 1 \
      -follow-redirects \
      -status-code \
      -content-type \
      -location \
      -response-time \
      -no-color \
      -silent \
      -json \
      -o recon/live/live_unauth.jsonl
  ```

- [ ] Filter to interesting status codes (200, 301, 302, 307, 308, 400, 403, 500):
  ```bash
  jq -c 'select(.status_code | . == 200 or . == 301 or . == 302
          or . == 307 or . == 308 or . == 400 or . == 403 or . == 500)' \
      recon/live/live_unauth.jsonl > recon/live/live_filtered_unauth.jsonl

  echo "Live (unauth): $(wc -l < recon/live/live_filtered_unauth.jsonl)"
  ```

- [ ] Extract live URLs for next stage:
  ```bash
  jq -r '.url' recon/live/live_filtered_unauth.jsonl \
      > recon/live/live_urls_unauth.txt
  ```

### 4B — Authenticated Probe

- [ ] Repeat with session cookie:
  ```bash
  AUTH_COOKIE=$(cat recon/auth_cookie.txt)

  httpx -l recon/processed/candidate_urls.txt \
      -threads 100 \
      -rate-limit 100 \
      -timeout 10 \
      -retries 1 \
      -follow-redirects \
      -status-code \
      -content-type \
      -location \
      -response-time \
      -H "Cookie: $AUTH_COOKIE" \
      -no-color \
      -silent \
      -json \
      -o recon/live/live_auth.jsonl
  ```

- [ ] Filter and extract same as 4A:
  ```bash
  jq -c 'select(.status_code | . == 200 or . == 301 or . == 302
          or . == 307 or . == 308 or . == 400 or . == 403 or . == 500)' \
      recon/live/live_auth.jsonl > recon/live/live_filtered_auth.jsonl

  jq -r '.url' recon/live/live_filtered_auth.jsonl \
      > recon/live/live_urls_auth.txt
  ```

- [ ] Compare auth vs unauth to find auth-gated endpoints:
  ```bash
  comm -13 \
      <(sort recon/live/live_urls_unauth.txt) \
      <(sort recon/live/live_urls_auth.txt) \
      > recon/live/auth_only_urls.txt
  echo "Auth-only endpoints: $(wc -l < recon/live/auth_only_urls.txt)"
  ```

---

## Stage 5 — Reflection Detection

For each live candidate URL: fetch it, extract the known variable segment value,
check if that value appears verbatim in the response body or headers.

This is the core recon step. The segment value from the Wayback URL is the test string.
No injection — you're checking whether the real historical value reflects.

### 5A — Build the Input Map

Join live URLs back to their candidate metadata (segment value + position):

```bash
python3 - <<'EOF'
import json

# Build lookup: url -> candidate metadata
candidates = {}
with open("recon/processed/candidates.jsonl") as f:
    for line in f:
        c = json.loads(line.strip())
        candidates[c["url"]] = c

# Write per-track input files (unauth and auth share same candidates,
# but we run against each track's live URL list)
for track, live_file, out_file in [
    ("unauth", "recon/live/live_urls_unauth.txt", "recon/live/reflection_input_unauth.jsonl"),
    ("auth",   "recon/live/live_urls_auth.txt",   "recon/live/reflection_input_auth.jsonl"),
]:
    with open(live_file) as lf, open(out_file, "w") as of:
        for line in lf:
            url = line.strip()
            if url in candidates:
                entry = candidates[url].copy()
                entry["track"] = track
                of.write(json.dumps(entry) + "\n")
EOF
```

### 5B — Async Reflection Check Script

Save this as `recon/scripts/check_reflection.py`:

```python
#!/usr/bin/env python3
"""
check_reflection.py — async path segment reflection checker

Usage:
  python3 check_reflection.py <input.jsonl> <output.jsonl> [--cookie "..."] [--header "..."]

Input JSONL fields required per line:
  url, segment_value, variable_position, pattern, cluster_size

Output JSONL adds:
  reflected_in_body (bool), reflected_in_headers (bool),
  header_names (list), status_code (int), content_type (str)
"""

import asyncio, aiohttp, json, sys, argparse
from urllib.parse import unquote

CONCURRENCY   = 80        # simultaneous requests — tune to target rate limits
TIMEOUT_S     = 12        # per-request timeout
MIN_SEG_LEN   = 5         # skip reflection check for segment values shorter than this

async def check(session, entry, results, sem):
    url       = entry["url"]
    seg_raw   = entry.get("segment_value", "")
    seg_dec   = unquote(seg_raw)

    if len(seg_dec) < MIN_SEG_LEN:
        return

    async with sem:
        try:
            async with session.get(url, allow_redirects=True, timeout=aiohttp.ClientTimeout(total=TIMEOUT_S)) as r:
                body = await r.text(errors="replace")
                headers = dict(r.headers)
                status  = r.status
                ct      = headers.get("Content-Type", "")

                in_body    = seg_dec.lower() in body.lower()
                header_hits = [k for k, v in headers.items()
                               if seg_dec.lower() in v.lower()]
                in_headers = len(header_hits) > 0

                if in_body or in_headers:
                    result = entry.copy()
                    result.update({
                        "reflected_in_body":    in_body,
                        "reflected_in_headers": in_headers,
                        "header_names":         header_hits,
                        "status_code":          status,
                        "content_type":         ct,
                    })
                    results.append(result)
        except Exception as e:
            pass  # dead host, timeout, SSL error — silently skip

async def main(input_file, output_file, extra_headers):
    entries = []
    with open(input_file) as f:
        for line in f:
            try:
                entries.append(json.loads(line.strip()))
            except Exception:
                pass

    print(f"[*] Checking {len(entries)} URLs for reflection...", file=sys.stderr)
    results = []
    sem     = asyncio.Semaphore(CONCURRENCY)

    connector = aiohttp.TCPConnector(ssl=False, limit=CONCURRENCY)
    async with aiohttp.ClientSession(headers=extra_headers, connector=connector) as session:
        tasks = [check(session, e, results, sem) for e in entries]
        await asyncio.gather(*tasks)

    with open(output_file, "w") as f:
        for r in results:
            f.write(json.dumps(r) + "\n")

    print(f"[+] Reflected: {len(results)} / {len(entries)}", file=sys.stderr)

if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("input")
    ap.add_argument("output")
    ap.add_argument("--cookie",  default="", help="Cookie header value")
    ap.add_argument("--header",  action="append", default=[],
                    help="Extra header as 'Name: value' (repeatable)")
    args = ap.parse_args()

    hdrs = {}
    if args.cookie:
        hdrs["Cookie"] = args.cookie
    for h in args.header:
        name, _, val = h.partition(":")
        hdrs[name.strip()] = val.strip()

    asyncio.run(main(args.input, args.output, hdrs))
```

```bash
mkdir -p recon/scripts
# save the script above to recon/scripts/check_reflection.py
pip3 install aiohttp --quiet
```

### 5C — Run Unauthenticated Reflection Check

- [ ] Install aiohttp if not present:
  ```bash
  pip3 install aiohttp
  ```

- [ ] Run:
  ```bash
  python3 recon/scripts/check_reflection.py \
      recon/live/reflection_input_unauth.jsonl \
      recon/reflected/reflected_unauth.jsonl
  ```

- [ ] Check hit count:
  ```bash
  echo "Reflected (unauth): $(wc -l < recon/reflected/reflected_unauth.jsonl)"
  ```

### 5D — Run Authenticated Reflection Check

- [ ] Run with cookie:
  ```bash
  python3 recon/scripts/check_reflection.py \
      recon/live/reflection_input_auth.jsonl \
      recon/reflected/reflected_auth.jsonl \
      --cookie "$(cat recon/auth_cookie.txt)"
  ```

  Or with Bearer token:
  ```bash
  python3 recon/scripts/check_reflection.py \
      recon/live/reflection_input_auth.jsonl \
      recon/reflected/reflected_auth.jsonl \
      --header "Authorization: Bearer $(cat recon/auth_header.txt)"
  ```

---

## Stage 6 — Post-Processing & Prioritization

Rank and deduplicate results before manual triage.

### 6A — Merge Tracks

- [ ] Merge auth and unauth results, tag source:
  ```bash
  python3 - <<'EOF'
  import json

  results = []
  for track, fname in [
      ("unauth", "recon/reflected/reflected_unauth.jsonl"),
      ("auth",   "recon/reflected/reflected_auth.jsonl"),
  ]:
      try:
          with open(fname) as f:
              for line in f:
                  r = json.loads(line.strip())
                  r["track"] = track
                  results.append(r)
      except FileNotFoundError:
          pass

  with open("recon/reflected/all_reflected.jsonl", "w") as f:
      for r in results:
          f.write(json.dumps(r) + "\n")

  print(f"Total reflected hits: {len(results)}")
  EOF
  ```

### 6B — Priority Ranking

Rank by reflection location (header > body) and cluster size (bigger cluster = more users affected):

```bash
python3 - <<'EOF'
import json

with open("recon/reflected/all_reflected.jsonl") as f:
    results = [json.loads(l) for l in f]

def priority(r):
    score = 0
    if r.get("reflected_in_headers"):
        score += 100   # header reflection = open redirect / CRLF potential
    if r.get("reflected_in_body"):
        score += 50
    score += min(r.get("cluster_size", 1), 50)  # larger clusters = more impactful
    return score

results.sort(key=priority, reverse=True)

with open("recon/reflected/ranked_results.jsonl", "w") as f:
    for r in results:
        f.write(json.dumps(r) + "\n")

# Also write a human-readable TSV
with open("recon/reflected/ranked_results.tsv", "w") as f:
    f.write("priority\ttrack\turl\tpattern\tsegment_value\tbody\theaders\theader_names\tstatus\n")
    for r in results:
        f.write("\t".join([
            str(priority(r)),
            r.get("track",""),
            r.get("url",""),
            r.get("pattern",""),
            r.get("segment_value",""),
            str(r.get("reflected_in_body",False)),
            str(r.get("reflected_in_headers",False)),
            ",".join(r.get("header_names",[])),
            str(r.get("status_code","")),
        ]) + "\n")

print(f"Ranked {len(results)} results → recon/reflected/ranked_results.tsv")
EOF
```

### 6C — Deduplicate by Pattern

One result per URL pattern is enough for recon — you don't need 10,000 URLs of the same template:

```bash
python3 - <<'EOF'
import json

seen_patterns = set()
unique = []
with open("recon/reflected/ranked_results.jsonl") as f:
    for line in f:
        r = json.loads(line.strip())
        key = (r.get("pattern",""), r.get("track",""), r.get("reflected_in_headers",""))
        if key not in seen_patterns:
            seen_patterns.add(key)
            unique.append(r)

with open("recon/reflected/unique_patterns.jsonl", "w") as f:
    for r in unique:
        f.write(json.dumps(r) + "\n")

print(f"Unique patterns with reflection: {len(unique)}")
EOF
```

### 6D — Known False-Positive Filters

- [ ] Remove hits where the segment value is a generic short word that commonly appears in pages:
  ```bash
  python3 - <<'EOF'
  import json, re

  # These segment values reflect trivially in boilerplate and are almost never exploitable
  BORING = {"true","false","null","none","home","page","list","view","new","all","any","top","get","set","put","add","del","run","log"}

  with open("recon/reflected/unique_patterns.jsonl") as f:
      results = [json.loads(l) for l in f]

  filtered = [r for r in results if r.get("segment_value","").lower() not in BORING]

  with open("recon/reflected/final_candidates.jsonl", "w") as f:
      for r in filtered:
          f.write(json.dumps(r) + "\n")

  print(f"Final candidates after FP filter: {len(filtered)}")
  EOF
  ```

---

## Stage 7 — Export to Burp Suite for Manual Triage

- [ ] Extract final URL list for Burp:
  ```bash
  jq -r '.url' recon/reflected/final_candidates.jsonl \
      > recon/burp/urls_to_verify.txt

  echo "URLs to manually verify in Burp: $(wc -l < recon/burp/urls_to_verify.txt)"
  ```

- [ ] In Burp Suite Pro:
  - [ ] Target > Site Map > right-click > "Add to scope" for the target domain(s)
  - [ ] Repeater: paste each URL from `urls_to_verify.txt` and manually confirm:
    - Is the segment value actually in the response (not a coincidental substring match)?
    - What is the HTML context? (raw HTML, inside a JS string, in an attribute, in a header value?)
    - Is it URL-encoded / HTML-encoded, or raw?
  - [ ] Tag in Burp: `reflected-raw` (no encoding), `reflected-encoded` (encoded — lower priority)

- [ ] For header-reflected URLs, check in Burp:
  - [ ] Is it in `Location:` → Open Redirect candidate
  - [ ] Is it in another header with newline chars possible → CRLF candidate
  - [ ] Is it in `Set-Cookie:` → Cookie injection candidate

---

## Stage 8 — Quick-Win Checks (Parallel to Stage 6 Review)

These can run while you're reviewing Stage 6 output.

### 8A — CRLF Candidates

Check which Wayback URLs already contain encoded newlines in path segments
(these are historical probes or real injection attempts, already confirmed to have been sent):

```bash
grep -E '%0[dD]|%0[aA]|\r|\n' recon/processed/candidate_urls.txt \
    > recon/reflected/crlf_candidates.txt
echo "URLs with encoded newlines in path: $(wc -l < recon/reflected/crlf_candidates.txt)"
```

### 8B — Open Redirect Candidates (Header Reflection)

```bash
jq -r 'select(.reflected_in_headers == true) | .url' \
    recon/reflected/final_candidates.jsonl \
    > recon/reflected/header_reflected_urls.txt
```

### 8C — Script Context Check

For body-reflected URLs, check if the reflection falls inside a `<script>` block:

```bash
python3 - <<'EOF'
import json, aiohttp, asyncio, re

async def check_script_ctx(session, entry, results, sem):
    url = entry["url"]
    seg = entry.get("segment_value", "")
    async with sem:
        try:
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as r:
                body = await r.text(errors="replace")
                # Find all <script>...</script> blocks
                scripts = re.findall(r'<script[^>]*>(.*?)</script>', body, re.DOTALL | re.IGNORECASE)
                in_script = any(seg.lower() in s.lower() for s in scripts)
                if in_script:
                    result = entry.copy()
                    result["in_script_block"] = True
                    results.append(result)
        except Exception:
            pass

async def main():
    with open("recon/reflected/final_candidates.jsonl") as f:
        body_hits = [json.loads(l) for l in f if json.loads(l).get("reflected_in_body")]

    results = []
    sem = asyncio.Semaphore(60)
    connector = aiohttp.TCPConnector(ssl=False, limit=60)
    async with aiohttp.ClientSession(connector=connector) as session:
        await asyncio.gather(*[check_script_ctx(session, e, results, sem) for e in body_hits])

    with open("recon/reflected/script_context_hits.jsonl", "w") as f:
        for r in results:
            f.write(json.dumps(r) + "\n")
    print(f"Reflected inside <script> blocks: {len(results)}")

asyncio.run(main())
EOF
```

---

## Checklist Summary — Run Order

```
Stage 0  — Setup scope + auth credentials
Stage 1  — gau URL collection
Stage 2A — Extension / path filter
Stage 2B — uro dedup
Stage 2C — Segment extraction (Python)
Stage 3A — Build static segment blocklist
Stage 3B — Cluster & score variable positions (Python)
Stage 4A — httpx live probe (unauth)
Stage 4B — httpx live probe (auth)          [if in scope]
Stage 5A — Install aiohttp
Stage 5B — Save check_reflection.py
Stage 5C — Run reflection check (unauth)
Stage 5D — Run reflection check (auth)      [if in scope]
Stage 6A — Merge tracks
Stage 6B — Rank results
Stage 6C — Dedup by pattern
Stage 6D — FP filter
Stage 7  — Export to Burp, manual triage
Stage 8  — CRLF / redirect / script-ctx quick-win checks
```

---

## Output Files Reference

| File                                        | Contents                                      |
| ------------------------------------------- | --------------------------------------------- |
| `recon/raw/gau_raw.txt`                     | All Wayback URLs, raw                         |
| `recon/processed/uro_deduped.txt`           | Deduplicated URL set                          |
| `recon/processed/candidates.jsonl`          | URLs with variable path segments + metadata   |
| `recon/live/live_filtered_unauth.jsonl`     | Live unauth responses (httpx)                 |
| `recon/live/live_filtered_auth.jsonl`       | Live auth responses (httpx)                   |
| `recon/reflected/all_reflected.jsonl`       | All reflection hits, both tracks              |
| `recon/reflected/ranked_results.tsv`        | Human-readable ranked results                 |
| `recon/reflected/final_candidates.jsonl`    | Deduped, FP-filtered final list               |
| `recon/burp/urls_to_verify.txt`             | URL list for Burp manual triage               |
| `recon/reflected/header_reflected_urls.txt` | Header-reflected (redirect / CRLF candidates) |
| `recon/reflected/script_context_hits.jsonl` | Body-reflected inside `<script>` blocks       |

---

*Checklist version: 2026-05-04. Scope: recon only — no payload injection.*
