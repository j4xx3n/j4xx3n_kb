# Hidden Parameter Discovery Checklist

> **Goal:** Surface parameters that exist in the target but are invisible to standard URL collection and fuzzing — hidden form fields, source maps, cookie-derived parameters, server-side header toggles, polluted duplicate parameters, and undocumented JSON properties. Input is the output of the Parameter Discovery phase: `live-params.txt`, `fuzzing/endpoints/live-all.txt`, `fuzzing/endpoints/tier1.txt`, `js/js-urls.txt`, and `fuzzing/merged-params.txt`.

---

## Pre-Flight

- [ ] Confirm all inputs from the parameter phase are present:
  ```bash
  wc -l fuzzing/endpoints/live-all.txt \
          fuzzing/endpoints/tier1.txt \
          js/js-urls.txt \
          live-params.txt \
          fuzzing/merged-params.txt
  ```
- [ ] Create working directories:
  ```bash
  mkdir -p hidden/{source,sourcemaps,cookies,headers,probe,hpp,json,redirects,reuse,wordlists}
  ```

---

## Step 1 — HTML Source Inspection

JavaScript analysis in the previous phase found parameters referenced in `.js` files. This step finds parameters embedded directly in HTML itself — hidden form fields, `name` and `id` attributes across every element, developer comments, and client-side config objects injected into `<script>` tags. None of these appear in any URL or archive.

### Extract hidden form fields

```bash
# Fetch each live page and pull every <input type="hidden"> element
while IFS= read -r url; do
  curl -sk "$url" 2>/dev/null \
    | grep -oiE '<input[^>]+type=["\x27]?hidden["\x27]?[^>]*>' \
    >> hidden/source/hidden-fields-raw.txt
done < fuzzing/endpoints/live-all.txt

# Pull just the name= values
grep -oiE 'name="[^"]+"|name='"'"'[^'"'"']+'"'"'' hidden/source/hidden-fields-raw.txt \
  | grep -oiE '"[^"]+"|'"'"'[^'"'"']+'"'"'' \
  | tr -d '"'"'" \
  | sort -u > hidden/source/hidden-field-names.txt

wc -l hidden/source/hidden-field-names.txt
# Review hidden-fields-raw.txt manually — value= attributes often contain tokens or state
```

### Scrape all id and name attributes from every HTML element

Elements beyond `<input>` — `<select>`, `<textarea>`, `<button>`, `<form>` — also carry `name` and `id` attributes the application reads as parameters:

```bash
while IFS= read -r url; do
  curl -sk "$url" 2>/dev/null \
    | grep -oiE '(id|name)="[a-zA-Z][a-zA-Z0-9_-]+"' \
    | grep -oiE '"[a-zA-Z][a-zA-Z0-9_-]+"' \
    | tr -d '"'
done < fuzzing/endpoints/live-all.txt \
  | sort -u > hidden/source/all-attr-names.txt

wc -l hidden/source/all-attr-names.txt
```

### Mine HTML comments

Developers routinely comment out debug flags, old parameter names, and internal API paths:

```bash
while IFS= read -r url; do
  curl -sk "$url" 2>/dev/null \
    | grep -oE '<!--.*?-->' \
    | sed "s|^|[$url] |" \
    >> hidden/source/html-comments.txt
done < fuzzing/endpoints/live-all.txt

wc -l hidden/source/html-comments.txt
# Review manually — look for ?param=, API routes, and key=value patterns
```

### Hunt for embedded JS config objects

Many web apps inject server-side configuration into the page as a global JS object (`window.__config`, `window.APP_DATA`, `window.initialState`). These objects contain parameter names, feature flags, and internal routes that never appear in any standalone JS file:

```bash
while IFS= read -r url; do
  curl -sk "$url" 2>/dev/null \
    | grep -oiE 'window\.[a-zA-Z_][a-zA-Z0-9_]{1,40}\s*=\s*\{[^;]{0,600}' \
    | sed "s|^|[$url] |" \
    >> hidden/source/js-config-objects.txt
done < fuzzing/endpoints/live-all.txt

wc -l hidden/source/js-config-objects.txt
# Extract key names from these objects and add to the Step 8 reuse wordlist
```

---

## Step 2 — Source Map Extraction

Source maps (`.js.map`) link minified production JavaScript back to the original readable source. When left publicly accessible, they expose full file paths, function names, parameter names, and internal API routes that minification was supposed to obscure. Test every collected `.js` URL for a corresponding `.map` file.

### Probe for accessible source maps

```bash
while IFS= read -r jsurl; do
  mapurl="${jsurl}.map"
  code=$(curl -sk -o /dev/null -w "%{http_code}" "$mapurl")
  if [ "$code" = "200" ]; then
    echo "$mapurl" >> hidden/sourcemaps/found-maps.txt
    curl -sk "$mapurl" \
      -o "hidden/sourcemaps/$(echo "$mapurl" | md5sum | cut -c1-8).map"
  fi
done < js/js-urls.txt

wc -l hidden/sourcemaps/found-maps.txt
```

### Extract parameter names and routes from source maps

```bash
# sourcesContent inside the .map JSON contains the full pre-minified source
for mapfile in hidden/sourcemaps/*.map; do
  jq -r '.sourcesContent[]? // empty' "$mapfile" 2>/dev/null \
    | grep -oiE '[?&][a-zA-Z][a-zA-Z0-9_-]{1,30}=' \
    | tr -d '?&=' \
    >> hidden/sourcemaps/sourcemap-params.txt

  # Also extract API route strings
  jq -r '.sourcesContent[]? // empty' "$mapfile" 2>/dev/null \
    | grep -oiE '"(/[a-zA-Z0-9/_-]{2,60})"' \
    | tr -d '"' \
    >> hidden/sourcemaps/sourcemap-routes.txt
done

sort -u hidden/sourcemaps/sourcemap-params.txt \
  > hidden/sourcemaps/sourcemap-params-dedup.txt
sort -u hidden/sourcemaps/sourcemap-routes.txt \
  > hidden/sourcemaps/sourcemap-routes-dedup.txt

wc -l hidden/sourcemaps/sourcemap-params-dedup.txt
wc -l hidden/sourcemaps/sourcemap-routes-dedup.txt
# Validate any new routes with httpx and add live ones to fuzzing/endpoints/tier1.txt
```

---

## Step 3 — Cookie & Header Parameter Analysis

Cookie names and custom HTTP headers are two parameter surfaces that never appear in URLs and are invisible to every tool in the previous phase.

### Extract cookie names and test them as parameters

Applications frequently accept the same name as both a cookie and a GET/POST parameter. Harvesting `Set-Cookie` names gives a targeted wordlist of high-probability hidden params:

```bash
# Harvest all Set-Cookie names from live responses
httpx -l fuzzing/endpoints/live-all.txt \
  -silent \
  -include-response-header \
  2>/dev/null \
  | grep -i "Set-Cookie" \
  | grep -oiE 'Set-Cookie:\s*[a-zA-Z0-9_-]+' \
  | awk '{print $2}' \
  | sort -u > hidden/cookies/cookie-names.txt

wc -l hidden/cookies/cookie-names.txt

# Fuzz cookie names as GET/POST params against Tier 1 endpoints
arjun -i fuzzing/endpoints/tier1.txt \
  -w hidden/cookies/cookie-names.txt \
  -oJ hidden/cookies/arjun-cookie-params.json
```

### Param Miner — header and cookie discovery modes

These modes run a fundamentally different scan from the GET/POST fuzzing in `3-PARAMETERS` — they target request headers and cookies, not URL or body parameters:

- [ ] In Burp: send a Tier 1 request to **Repeater**
- [ ] Right-click → **Extensions → Param Miner → Guess headers** — finds hidden server-processing headers (`X-Debug`, `X-Admin`, routing headers)
- [ ] Right-click → **Extensions → Param Miner → Guess cookie parameters** — finds undocumented cookies the server reads
- [ ] Repeat for every distinct endpoint type in `tier1.txt` (login, search, account, admin, API)
- [ ] Save all findings to `hidden/headers/paramminer-headers.txt`

### Manual header probing

A curated set of headers known to toggle debug behavior, bypass access controls, or change server routing. Any response that differs from the baseline is worth investigating:

```bash
# Create the header wordlist
cat << 'EOF' > hidden/wordlists/headers.txt
X-Debug: true
X-Debug: 1
X-Admin: true
X-Admin: 1
X-Internal: true
X-Internal: 1
X-Test: true
X-Dev-Mode: true
X-Backend: true
X-Feature-Flag: debug
X-Forwarded-For: 127.0.0.1
X-Real-IP: 127.0.0.1
X-Originating-IP: 127.0.0.1
X-Custom-IP-Authorization: 127.0.0.1
X-Original-URL: /admin
X-Rewrite-URL: /admin
X-Override-URL: /admin
EOF

# Probe — flag any response that differs from the no-header baseline
while IFS= read -r url; do
  baseline=$(curl -sk -o /dev/null -w "%{http_code}:%{size_download}" "$url")
  while IFS= read -r header; do
    result=$(curl -sk -o /dev/null -w "%{http_code}:%{size_download}" -H "$header" "$url")
    if [ "$result" != "$baseline" ]; then
      echo "DIFF | $url | [$header] | baseline=$baseline → $result" \
        >> hidden/headers/header-probe-diffs.txt
    fi
  done < hidden/wordlists/headers.txt
done < fuzzing/endpoints/tier1.txt

wc -l hidden/headers/header-probe-diffs.txt
```

---

## Step 4 — Manual High-Value Parameter Probing

Generic wordlists contain tens of thousands of names but are diluted with noise. This step uses a curated set of high-signal debug, admin, and feature-flag parameter pairs — the kind that ship accidentally in production and are missed by brute-force tools because their values matter as much as their names. Run these specifically against 403 pages and empty-200 endpoints, which have the highest yield for bypass params.

### Build the high-value param wordlist

```bash
cat << 'EOF' > hidden/wordlists/high-value-params.txt
debug=true
debug=1
debug=on
admin=true
admin=1
test=true
test=1
internal=true
internal=1
preview=true
preview=1
beta=true
beta=1
enable=1
enabled=true
verbose=1
verbose=true
trace=1
trace=true
log=1
log=true
dev=true
dev=1
format=json
format=xml
format=debug
mode=debug
mode=admin
mode=test
mode=dev
access=full
access=admin
role=admin
privilege=1
override=1
show=all
show=hidden
v=2
version=2
ver=2
callback=test
json=1
xml=1
EOF
```

### Probe against 403 and empty-200 endpoints

403 pages and empty-200 responses are the most likely to unlock behavior when a hidden param is supplied. Flag any response that differs from the baseline:

```bash
# Build high-priority probe targets: 403s + empty-200s from 3-PARAMETERS
httpx -l fuzzing/endpoints/live-all.txt \
  -silent -mc 403 \
  > hidden/probe/403-targets.txt

cat hidden/probe/403-targets.txt \
    fuzzing/endpoints/empty-200.txt \
  | sort -u > hidden/probe/high-priority-targets.txt

wc -l hidden/probe/high-priority-targets.txt

# Probe — flag any response that differs from baseline
while IFS= read -r url; do
  baseline=$(curl -sk -o /dev/null -w "%{http_code}:%{size_download}" "$url")
  while IFS= read -r param; do
    result=$(curl -sk -o /dev/null -w "%{http_code}:%{size_download}" "${url}?${param}")
    if [ "$result" != "$baseline" ]; then
      echo "DIFF | ${url}?${param} | baseline=$baseline → $result" \
        >> hidden/probe/manual-probe-diffs.txt
    fi
  done < hidden/wordlists/high-value-params.txt
done < hidden/probe/high-priority-targets.txt

wc -l hidden/probe/manual-probe-diffs.txt
# Every line here is a potential finding — investigate each diff manually in Burp
```

---

## Step 5 — HTTP Parameter Pollution (HPP)

Sending duplicate parameter names exploits the fact that no HTTP standard defines how servers should handle them. Different backends parse them differently — some use the first value, some the last, some concatenate. A WAF reading one occurrence while the application processes another enables filter bypass, authentication bypass, and XSS.

### Fingerprint backend parser behavior

Before testing, determine how the target resolves duplicate parameters. Use a known working parameter from `live-params.txt`:

```bash
# Send both a legitimate value and a canary value for the same parameter
# Replace target.com/endpoint, 'id', and '1' with real values from live-params.txt
curl -sk "https://target.com/endpoint?id=LEGIT&id=HPP_CANARY" \
  | grep -oE "LEGIT|HPP_CANARY|LEGIT,HPP_CANARY"

# LEGIT             → first-occurrence (JSP / Tomcat)
# HPP_CANARY        → last-occurrence (PHP / Django / Rails)
# LEGIT,HPP_CANARY  → concatenation (ASP.NET / IIS)
# no match          → duplicates rejected or stripped
```

Server behavior reference:

| Backend | Parses `?p=1&p=2` as |
|---|---|
| ASP.NET / IIS | `1,2` (concatenated with comma) |
| PHP | `2` (last value wins) |
| JSP / Tomcat | `1` (first value wins) |
| Python / Django | `2` (last value wins) |
| Ruby / Rails | `2` (last value wins) |
| Node.js / Express | `['1','2']` (array) |

### Server-side HPP — GET

For each URL in `live-params.txt`, append a duplicate of each existing parameter with a canary value and flag responses that differ from the baseline:

```bash
while IFS= read -r url; do
  base=$(echo "$url" | cut -d'?' -f1)
  query=$(echo "$url" | grep -o '?.*' | cut -c2-)

  echo "$query" | tr '&' '\n' | while IFS= read -r pair; do
    key=$(echo "$pair" | cut -d'=' -f1)
    [ -z "$key" ] && continue

    baseline=$(curl -sk -o /dev/null -w "%{http_code}:%{size_download}" "$url")
    polluted=$(curl -sk -o /dev/null -w "%{http_code}:%{size_download}" \
      "${url}&${key}=HPP_CANARY")

    if [ "$polluted" != "$baseline" ]; then
      echo "HPP_DIFF | param=${key} | baseline=$baseline → $polluted | $url" \
        >> hidden/hpp/hpp-get-results.txt
    fi
  done
done < live-params.txt

wc -l hidden/hpp/hpp-get-results.txt
```

### Client-side HPP — reflective pages

Pages that reflect a parameter value back in HTML (search results, error messages) are vulnerable to client-side HPP where a polluted parameter injects content into a URL or form attribute:

- [ ] In `live-params.txt`, identify pages that reflect a parameter value back in the HTML response
- [ ] For each reflective parameter, append `%26HPP_TEST` to the value: `?q=test%26HPP_TEST`
- [ ] In Burp, inspect the response for `&HPP_TEST` or `&amp;HPP_TEST` appearing inside an HTML attribute (`href=`, `src=`, `action=`, `data-url=`)
- [ ] A match inside an attribute means the URL construction is pollutable — escalate to open redirect or XSS testing

---

## Step 6 — JSON Mass Assignment

Web apps that accept JSON request bodies often process undocumented object properties if they match internal data model fields. This is Mass Assignment — the app accepts a property it was never intended to expose (e.g., `"role"`, `"isAdmin"`, `"price"`) because the ORM or framework maps all incoming JSON keys to the model automatically.

### Param Miner — JSON mode (Burp Suite)

- [ ] Intercept a POST request with a JSON body (login, account update, API call)
- [ ] Right-click → **Extensions → Param Miner → Guess JSON parameter**
- [ ] Monitor the **Output** tab for `Identified parameter` entries
- [ ] In Burp Pro: check **Dashboard → Issues Activity** for `Secret Input` flags
- [ ] Save results to `hidden/json/paramminer-json.txt`

### x8 — JSON body fuzzing

```bash
# Fuzz for undocumented JSON properties on all Tier 1 endpoints
while IFS= read -r url; do
  x8 -u "$url" \
    -X POST \
    --data '{}' \
    -H "Content-Type: application/json" \
    -w fuzzing/merged-params.txt \
    -o "hidden/json/x8-json-$(echo "$url" | md5sum | cut -c1-8).txt" 2>/dev/null
done < fuzzing/endpoints/tier1.txt
```

### High-value JSON property probing

Manually test properties that generic wordlists miss — internal model fields commonly exposed by ORMs and frameworks:

```bash
cat << 'EOF' > hidden/wordlists/json-mass-assign.txt
role
isAdmin
is_admin
admin
privilege
permissions
group
access
status
active
enabled
verified
confirmed
price
cost
amount
discount
balance
credit
internal
debug
bypass
override
EOF

# Probe each property against every Tier 1 endpoint
while IFS= read -r url; do
  while IFS= read -r prop; do
    result=$(curl -sk -X POST "$url" \
      -H "Content-Type: application/json" \
      -d "{\"${prop}\": true}" \
      -o /dev/null -w "%{http_code}")
    echo "$result $url property=${prop}"
  done < hidden/wordlists/json-mass-assign.txt
done < fuzzing/endpoints/tier1.txt \
  | grep -vE "^(404|400|405|415) " \
  >> hidden/json/json-probe-results.txt

wc -l hidden/json/json-probe-results.txt
```

---

## Step 7 — 3xx Response Body Inspection

When a server returns a 301 or 302, browsers discard the response body and follow the `Location` header. That body frequently leaks sensitive content — session tokens, internal paths, hidden parameters — because developers assume it is never seen. Capture it before the redirect is followed.

```bash
# Capture full response body for every 3xx without following the redirect
while IFS= read -r url; do
  response=$(curl -sk --max-redirs 0 -D - "$url" 2>/dev/null)
  code=$(echo "$response" | head -1 | grep -oE '[0-9]{3}')
  if echo "$code" | grep -qE "^3[0-9]{2}$"; then
    body=$(echo "$response" | sed '1,/^\r$/d')
    if [ "$(echo "$body" | tr -d '[:space:]' | wc -c)" -gt 10 ]; then
      echo "=== [$code] $url ===" >> hidden/redirects/redirect-bodies.txt
      echo "$body" >> hidden/redirects/redirect-bodies.txt
    fi
  fi
done < fuzzing/endpoints/live-all.txt

wc -l hidden/redirects/redirect-bodies.txt
```

Extract any parameter-looking strings from captured bodies:

```bash
grep -oiE '[?&][a-zA-Z][a-zA-Z0-9_-]{1,30}=[^&"'"'"'<> ]{0,100}' \
  hidden/redirects/redirect-bodies.txt \
  | sort -u > hidden/redirects/redirect-body-params.txt

wc -l hidden/redirects/redirect-body-params.txt
# Tokens and session values appearing here are immediate findings — review manually
```

---

## Step 8 — Cross-Endpoint Parameter Reuse

Parameters discovered on one endpoint are frequently accepted silently by others that never advertised them. A `user_id` param found on `/profile` may work on `/admin/edit`, `/api/export`, or `/checkout`. This step compiles every parameter name found across this entire file and the previous phase, then re-tests all of them against every Tier 1 endpoint.

### Compile master discovered-parameter list

```bash
cat hidden/source/hidden-field-names.txt \
    hidden/source/all-attr-names.txt \
    hidden/sourcemaps/sourcemap-params-dedup.txt \
    hidden/cookies/cookie-names.txt \
  | sort -u > hidden/reuse/this-phase-params.txt

# Merge with custom params built in 3-PARAMETERS
cat fuzzing/custom-params.txt \
    hidden/reuse/this-phase-params.txt \
  | sort -u > hidden/reuse/master-params.txt

wc -l hidden/reuse/master-params.txt
```

### Arjun — bulk cross-endpoint fuzzing

```bash
arjun -i fuzzing/endpoints/tier1.txt \
  -w hidden/reuse/master-params.txt \
  -oJ hidden/reuse/arjun-cross-endpoint.json
```

### x8 — parallel coverage

```bash
while IFS= read -r url; do
  x8 -u "$url" \
    -w hidden/reuse/master-params.txt \
    -o "hidden/reuse/x8-$(echo "$url" | md5sum | cut -c1-8).txt" 2>/dev/null
done < fuzzing/endpoints/tier1.txt
```

---

## Final Output

```bash
echo "=== Hidden Parameter Discovery Summary ==="
echo "Header probe diffs:       $(wc -l < hidden/headers/header-probe-diffs.txt 2>/dev/null || echo 0)"
echo "Manual param probe diffs: $(wc -l < hidden/probe/manual-probe-diffs.txt 2>/dev/null || echo 0)"
echo "HPP candidates:           $(wc -l < hidden/hpp/hpp-get-results.txt 2>/dev/null || echo 0)"
echo "Redirect body params:     $(wc -l < hidden/redirects/redirect-body-params.txt 2>/dev/null || echo 0)"
echo "Source map params:        $(wc -l < hidden/sourcemaps/sourcemap-params-dedup.txt 2>/dev/null || echo 0)"
echo "Master reuse wordlist:    $(wc -l < hidden/reuse/master-params.txt 2>/dev/null || echo 0)"
```

**Priority manual review files — each line is a potential finding:**
- `hidden/headers/header-probe-diffs.txt` — header toggles that changed server behavior
- `hidden/probe/manual-probe-diffs.txt` — debug/admin params that unlocked different responses
- `hidden/hpp/hpp-get-results.txt` — HPP candidates (verify parser behavior before escalating)
- `hidden/redirects/redirect-body-params.txt` — params and tokens leaked in redirect bodies
- `hidden/source/html-comments.txt` — HTML comments containing routes or param references
- `hidden/source/js-config-objects.txt` — embedded config objects with internal structure

**All confirmed parameters feed directly into attack-specific checklists.**

---

## Notes

<!-- Add confirmed hidden parameters, HPP parser fingerprint result, header findings, source map routes, and mass assignment discoveries here -->
