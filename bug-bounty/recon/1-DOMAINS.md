# Root Domain Discovery Checklist

> **Goal:** Find every root domain owned by the target company before touching subdomains. Work through all steps, collecting output into phase-specific files, then consolidate and verify. A domain that fails any check in Step 8 gets cut — do not test it.

---

## Step 1 — Gather Seed Data

Collect the minimum inputs needed before running any tools.

- [ ] **Company legal name** — exact registered name (e.g., "Acme Corporation" not "Acme")
- [ ] **Known in-scope root domain(s)** — pulled directly from the program scope tab
- [ ] **Registrant email pattern** — check WHOIS on a known domain:
  ```bash
  whois known-domain.com | grep -i "registrant\|email"
  ```
- [ ] **Known ASN(s)** — some programs list these directly in scope; note them now
- [ ] **Program scope page open** — keep it open for reference throughout; you'll check each find against it

Save all discovered domains to phase-specific `.txt` files as you go. Merge at Step 7.

---

## Step 2 — Reverse WHOIS Pivoting

Find all domains registered by the same email address or company name.

### GDPR Note
WHOIS records after May 2018 are often redacted. For redacted records: use nameservers (visible even when redacted) and pivot to pre-2018 historical records where available.

### By Registrant Email
- [ ] Run Amass intel mode against known domain to extract registrant info:
  ```bash
  amass intel -d known-domain.com -whois
  ```
- [ ] ViewDNS reverse WHOIS by email (web): `https://viewdns.info/reversewhois/`
- [ ] ReverseWhois.io (web, free): enter registrant email or company name
- [ ] WhoisXMLAPI (500 free monthly credits): `https://tools.whoisxmlapi.com/reverse-whois-search`
- [ ] Whoxy.com (largest free database, ~329M records): `https://www.whoxy.com/reverse-whois/`
- [ ] CLI shortcut with knockknock:
  ```bash
  knockknock -name "Company Name"
  ```

### By Company Name (Amass)
- [ ] Search Amass intel directly by org name:
  ```bash
  amass intel -org "Company Name" -o phase2-whois.txt
  ```

### Nameserver Correlation
If WHOIS is fully redacted, nameservers are still visible and are the most reliable pivot:
- [ ] Get nameservers for all known domains:
  ```bash
  for domain in known1.com known2.com; do echo "$domain: $(dig NS $domain +short)"; done
  ```
- [ ] Cross-reference: any discovered domain sharing the same nameservers is a high-confidence match

Save all output → `phase2-whois.txt`

---

## Step 3 — Certificate Transparency Logs

SSL certificates are public. Every domain that has ever had a cert issued is searchable.

### Search by Organization Name
- [ ] Query crt.sh by org name (finds certs where the org field matches):
  ```bash
  curl -s "https://crt.sh/?O=Company+Name&output=json" | jq -r '.[].common_name' | sort -u
  ```
- [ ] Search crt.sh by known domain wildcard to find related SAN entries:
  ```bash
  curl -s "https://crt.sh/?q=%25known-domain.com&output=json" | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u
  ```
- [ ] Amass intel via CT logs:
  ```bash
  amass intel -d known-domain.com -active -o phase3-ct.txt
  ```
- [ ] Censys web search (requires free account): search `parsed.subject.organization: "Company Name"` in the certificate index

### Time-Correlation Attack
When a server auto-renews certs (cPanel, Plesk), all hosted domains renew at the same second. Find related domains by timestamp:
- [ ] Get the cert issuance timestamp for a known domain from crt.sh
- [ ] On Censys, filter: `parsed.validity.start: <timestamp>` within a ±5 second window
- [ ] Any certs issued at the exact same moment share the same server → high-confidence related domains

Save all output → `phase3-ct.txt`

---

## Step 4 — ASN and IP Space Discovery

Find the company's IP ranges, then reverse-resolve hostnames out of them.

### Find the ASN
- [ ] Search BGP.he.net by company name: `https://bgp.he.net/` → look for matching org entries
- [ ] BGPView API:
  ```bash
  curl -s "https://api.bgpview.io/search?query_term=Company+Name" | jq '.data.asns[] | .asn, .description'
  ```
- [ ] Extract ASN from a known IP:
  ```bash
  curl -s "http://ip-api.com/json/$(dig +short known-domain.com | head -1)" | jq -r '.as'
  ```

### Extract CIDR Ranges from ASN
- [ ] Query RADB for all IP blocks belonging to the ASN:
  ```bash
  whois -h whois.radb.net -- '-i origin AS<NUMBER>' | grep -Eo '([0-9.]+){4}/[0-9]+' | sort -u
  ```
- [ ] Nmap ASN script (alternative):
  ```bash
  nmap --script targets-asn --script-args targets-asn.asn=<NUMBER>
  ```
- [ ] Check RIR directly for additional allocations:
  - ARIN (North America): `https://search.arin.net/rdap/?query=CompanyName`
  - RIPE (Europe): `https://apps.db.ripe.net/db-web-ui/query?searchtext=CompanyName`
  - APNIC (Asia-Pacific): `https://search.apnic.net/`

### Reverse DNS on IP Ranges
- [ ] Expand CIDR and run PTR lookups — pipe each range through mapcidr then dnsx:
  ```bash
  echo "1.2.3.0/24" | mapcidr -silent | dnsx -ptr -resp-only -o phase4-ptr.txt
  ```
- [ ] For multiple ranges:
  ```bash
  cat cidr-ranges.txt | mapcidr -silent | dnsx -ptr -resp-only >> phase4-ptr.txt
  ```

Save all output → `phase4-ptr.txt`

---

## Step 5 — Tracking ID and Favicon Correlation

Find domains the company forgot to link by the analytics code and favicon they share.

### Google Analytics ID
- [ ] View source on the known domain and find the GA ID:
  ```bash
  curl -s https://known-domain.com | grep -Eo '(UA-[0-9]+-[0-9]+|G-[A-Z0-9]+)'
  ```
- [ ] Enter that ID into SpyOnWeb: `https://spyonweb.com/`
- [ ] Cross-check on DNSlytics: `https://dnslytics.com/reverse-analytics`
- [ ] BuiltWith relationship profiles (finds shared GA, AdSense, Facebook Pixel IDs): `https://builtwith.com/relationships/known-domain.com`

### Favicon Hash (Shodan)
- [ ] Generate MurmurHash of the favicon:
  ```bash
  curl -s https://known-domain.com/favicon.ico | python3 -c \
    "import sys,mmh3,base64; d=sys.stdin.buffer.read(); print(mmh3.hash(base64.encodebytes(d)))"
  ```
- [ ] Search Shodan for all assets sharing that hash (requires free Shodan account):
  ```bash
  shodan search http.favicon.hash:<HASH> --fields ip_str,port,hostnames
  ```

Save all output → `phase5-tracking.txt`

---

## Step 6 — Auxiliary Methods

### Azure AD Tenant Enumeration
Works for any company using Microsoft 365 or Azure — reveals all verified domains in their tenant:
- [ ] Web tool (fastest): `https://aadinternals.com/osint/` — enter any known domain
- [ ] CLI check to confirm tenant exists before visiting:
  ```bash
  curl -s "https://login.microsoftonline.com/known-domain.com/v2.0/.well-known/openid-configuration" | jq -r '.issuer'
  ```
  A valid issuer URL confirms the domain is in an Azure tenant → use AADInternals to enumerate all tenant domains.

> **Limitation:** Only works if the company uses Microsoft services.

### Acquisitions and Subsidiaries
Companies acquire other businesses and inherit their domains — these are frequently forgotten and missed by other hunters:
- [ ] Search Crunchbase for the company → check the "Acquisitions" tab
- [ ] Wikipedia company page → look for subsidiaries/spin-offs section
- [ ] Google: `"Company Name" acquired site:techcrunch.com OR site:crunchbase.com`
- [ ] For each acquired company found, run Steps 2–5 against that entity's name as a new seed

### GitHub Recon
Internal domain names appear in source code, config files, and issue threads:
- [ ] Search GitHub by org name:
  ```bash
  # In GitHub web search:
  org:CompanyGitHubHandle known-domain.com
  ```
- [ ] Look for `.internal`, `.corp`, `.dev` domain patterns in code and Issues
- [ ] Tools for automated scanning:
  ```bash
  gitrob --github-access-token <TOKEN> CompanyGitHubHandle
  ```

Save all output → `phase6-aux.txt`

---

## Step 7 — Consolidate

Merge all phase outputs into a single deduplicated list:
```bash
cat phase2-whois.txt phase3-ct.txt phase4-ptr.txt phase5-tracking.txt phase6-aux.txt \
  | sed 's/\*\.//g' \
  | tr '[:upper:]' '[:lower:]' \
  | sort -u > all-discovered.txt

wc -l all-discovered.txt
```

Strip any entries that are clearly not root domains (bare IPs, subdomains you'll enumerate later):
```bash
grep -E '^[a-z0-9]([a-z0-9\-]{0,61}[a-z0-9])?(\.[a-z]{2,})+$' all-discovered.txt > root-domains.txt
```

---

## Step 8 — Verify Each Domain

Run every domain in `root-domains.txt` through all three checks. A domain must pass all three to make the final working list. Fail any one → cut it.

### Check A — In Scope

- [ ] Open the program scope tab
- [ ] The domain must appear explicitly, or fall under a wildcard (`*.company.com`) or a broad org-level scope statement
- [ ] Check the **out-of-scope list** — some programs explicitly exclude acquired brands or third-party properties
- [ ] If unclear, check the parent domain: if `company.com` is in scope and you found `company-brand.com` via WHOIS, confirm before testing

**Cut if:** Domain is not listed in scope and is not clearly covered by a wildcard → **SKIP**

### Check B — Live and Resolving

- [ ] Bulk DNS resolution — only keep domains with active A records:
  ```bash
  cat root-domains.txt | dnsx -silent -a -resp-only > live-domains.txt
  ```
- [ ] Domains with only MX records (mail servers) may still be valuable — check separately:
  ```bash
  cat root-domains.txt | dnsx -silent -mx -resp-only > mail-domains.txt
  ```

**Cut if:** No A, AAAA, or MX record resolves → **SKIP** (save to `dead-domains.txt` for monitoring; they may come live later)

### Check C — Actually Owned by the Target Company

For each live domain, confirm the registrant or hosting org is the target — not a CDN, vendor, or unrelated party:
```bash
whois <domain> | grep -iE 'org|organization|registrant name|registrant org'
```

| Signal | Action |
|---|---|
| Registrant org matches company name | **Confirmed — keep** |
| Domain resolves to company's known ASN | **Confirmed — keep** |
| Nameserver matches known company domains | **Likely owned — keep** |
| CNAME points to a known company domain | **Confirmed — keep** |
| Registrant is a privacy proxy service | Check historical WHOIS or other pivot before deciding |
| Hosted on CDN (Cloudflare/Fastly), registrant unclear | Run `whois` on the underlying IP; check BuiltWith |
| Registrant is clearly a different unrelated company | **Cut** |
| Domain is a vendor SaaS platform (e.g., Salesforce, Zendesk hosted subdomain) | **Cut** |

- [ ] Check each ambiguous domain:
  ```bash
  whois $(dig +short <domain> | tail -1) | grep -iE 'org|netname|owner'
  ```

**Cut if:** WHOIS org is a third party with no clear relationship to the target company → **SKIP**

---

## Step 9 — Final Working List

Domains that passed all three checks in Step 8 go here:
```bash
# Manually curate after Step 8
> verified-domains.txt
```

This list feeds directly into subdomain enumeration (next phase).

---

## Notes

<!-- Add discovered domains, pivot trails, and any unusual findings here -->
