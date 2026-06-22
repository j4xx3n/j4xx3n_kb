# Bug Bounty Program Selection Checklist  
  
> **Goal:** Find a paid bug bounty program with wide/wildcard scope that will reliably pay out for a beginner hunter. Work through every step in order. A single NO-GO in Step 2 ends evaluation — skip the program immediately.  
  
---  
  
## Step 1 — Discover Candidate Programs  
  
### Platform Browsing  
- [ ] Browse public programs on **HackerOne** — filter by "Bounty" type, sort by "Last updated"  
- [ ] Browse public programs on **Bugcrowd** — use CrowdMatch skill filters to surface relevant programs  
- [ ] Browse **Intigriti** — good for fast payout times and European-hosted targets  
- [ ] Browse **YesWeHack** — smaller competition pool than HackerOne/Bugcrowd  
  
### Aggregators and Discovery Tools  
- [ ] Use **[Firebounty](https://firebounty.com)** — filters by platform, payout level, scope size, and launch date across all major platforms at once  
- [ ] Subscribe to **new program announcement feeds** on HackerOne and Bugcrowd — being among the first 20 researchers on a new program dramatically improves odds  
  
### Google Dorking for Self-Hosted Programs  
These programs run outside major platforms and have far less competition:  
```  
inurl:/security  
inurl:security.txt  
inurl:/responsible-disclosure  
"bug bounty" inurl:security site:target.com  
```  
- [ ] For each self-hosted program found, verify it offers monetary rewards before proceeding  
  
### Program Age Filter  
- [ ] Note the program launch date  
 - **6–18 months old** = sweet spot — functional triage process, still enough untouched surface area → **CONTINUE**  
 - **Under 6 months** = process may be immature, but highest low-hanging fruit → **CONTINUE with caution**  
 - **Over 3 years old** = surface likely well-picked by experienced hunters → **SKIP unless scope recently expanded**  
 - **No activity in last 90 days** = dormant → **SKIP**  
  
---  
  
## Step 2 — Instant Disqualifiers (Hard No-Go Gates)  
  
Check every item below before doing anything else. **Any single NO-GO = skip this program entirely.**  
  
- [ ] **Safe harbor clause present?**  
 Explicit language protecting researchers from legal action for good-faith in-scope testing.  
 Missing → **NO-GO**  
  
- [ ] **NDA required to participate?**  
 Yes → **NO-GO** — prevents disclosure even after remediation  
  
- [ ] **90-day payout rate above 25%?**  
 Find this in platform statistics (HackerOne program page → "Statistics" tab; Bugcrowd → program metrics).  
 At or below 25% → **NO-GO**  
  
- [ ] **Time-to-first-response consistently under 2 weeks?**  
 Check platform-published response time metrics. If community reports (Reddit, Discord) show consistent 2+ week silences → **NO-  
GO**  
  
- [ ] **Scope covers meaningful application functionality?**  
 If most core features (auth, user data, payments, API) are listed as out-of-scope → **NO-GO**  
  
- [ ] **No active researcher complaints about bad-faith closures or dismissive triage?**  
 Search community sources (Step 6) first if unsure. Consistent pattern of complaints → **NO-GO**  
  
- [ ] **Program is not currently paused?**  
 Paused without announced restart date → **NO-GO**  
  
---  
  
## Step 3 — Scope Evaluation (Wide/Wildcard Focus)  
  
These are hard thresholds. Fail any one → **NO-GO**.  
  
### Wildcard Scope Check  
- [ ] **Wildcard domain present** (`*.domain.com` or multiple root domains)?  
 Single static URL only (e.g., `https://domain.com/app`) with no wildcards → **NO-GO**  
- [ ] **Preview the attack surface** — run a quick passive check before committing:  
 - Visit `crt.sh/?q=%.domain.com` to estimate subdomain count  
 - Under ~10 subdomains visible = narrow surface → reconsider  
 - 30+ subdomains = wide surface → **GO**  
  
### Scope Content Check  
- [ ] **Explicit out-of-scope list exists** — vague or catch-all exclusions give the company maximum discretion to reject valid r  
eports → **NO-GO** if absent  
- [ ] **Scope includes at least one of:** user authentication, account management, payment flows, API endpoints, or user data storage  
- [ ] **Testing credentials available** — either self-registration is allowed or the program provides test accounts. Neither avai  
lable → **NO-GO**  
- [ ] **Automation not completely prohibited** — scanners and crawlers allowed (even with rate-limit restrictions) → **GO** / Bla  
nket "no automated tools" policy with no exceptions → **FLAG**  
  
### Scope Bonus Signals (not required, but raise confidence)  
- [ ] API endpoints explicitly in scope  
- [ ] Mobile apps in scope (additional surface area)  
- [ ] Program explicitly mentions vulnerability classes they prioritize (signals where the security team's blind spots are)  
- [ ] Production environment testable (not staging-only)  
  
---  
  
## Step 4 — Financial Viability (Hard Thresholds)  
  
### Payout Rate  
- [ ] **90-day payout rate > 25%** → **GO** / ≤ 25% → **NO-GO**  
 - Find on HackerOne: program page → "Response Efficiency" stats  
 - Find on Bugcrowd: program metrics section  
  
### Payment Timeline  
- [ ] **Time-to-payment ≤ 30 days post-validation** → **GO**  
 - Best: ≤ 10 days (Intigriti-level)  
 - Acceptable: 10–30 days  
 - **NO-GO**: consistently over 3 months (visible in community reports)  
  
### Bounty Table  
- [ ] **Transparent bounty table published** (P1–P4 or Critical/High/Medium/Low tiers with amounts)?  
 No table at all → **FLAG** — ad-hoc decisions increase risk of unexplained reductions  
  
### Expected Value Sanity Check  
- [ ] Calculate: **average bounty × payout rate = expected value per valid report**  
 - Example: $800 avg × 40% payout rate = $320 expected value per report submitted  
 - If expected value feels too low relative to the effort the scope demands → skip  
  
---  
  
## Step 5 — Triage Quality (Hard Thresholds)  
  
### Response Time  
- [ ] **Time-to-first-response < 48 hours** (platform-published average) → **GO**  
 - Acceptable: 48 hours – 1 week  
 - **NO-GO**: consistently over 2 weeks  
  
### SLA Compliance  
- [ ] **SLA compliance ≥ 75%** → **GO** / Below 75% → **NO-GO**  
 - 90%+ = exceptional, strong signal of a well-managed program  
  
### Disclosed Reports Check  
- [ ] Browse the program's publicly disclosed reports (HackerOne "Hacktivity" feed / Bugcrowd disclosures)  
 - Program responses are substantive and technical → **GO**  
 - Reports closed with one-line "N/A" and no explanation → **FLAG**  
 - No public disclosures at all → **FLAG** (opacity about program health)  
 - Check whether the same vulnerability classes are repeatedly reported — exhausted surface is a signal to reconsider  
  
---  
  
## Step 6 — Community Due Diligence  
  
Research researcher sentiment before committing any time. Do all three.  
  
### Reddit  
- [ ] Search **r/bugbounty** for the program or company name  
 - Look for: triage complaints, payout delays, scope disputes, positive writeup mentions  
 - Net negative sentiment from multiple independent researchers → **NO-GO**  
  
### Discord  
- [ ] Search the major bug bounty Discord communities (e.g., Bugcrowd Discord, HackerOne community, NahamSec Discord) for the pro  
gram name  
 - Same signal criteria as Reddit  
  
### Twitter / X  
- [ ] Search `"[program name]" bug bounty` and `"[company]" bounty` on Twitter/X  
 - Public complaints from known researchers → **FLAG** or **NO-GO** depending on severity and recency  
  
### What You Are Looking For  
| Signal | Meaning |  
|---|---|  
| Multiple researchers report non-payment or extreme delays | NO-GO |  
| Triagers repeatedly misunderstand technical findings | NO-GO |  
| Reports closed as N/A with no explanation pattern | NO-GO |  
| Researchers mention fast payments and good communication | Strong GO |  
| Active writeups / disclosed reports linked in community | Strong GO |  
| Complaints are old (1+ years) and recent reports are positive | Proceed with caution |  
  
---  
  
## Step 7 — Competition and Saturation Check  
  
### Avoid Mega-Programs  
- [ ] Program is **not** run by a major consumer tech company (Google, Meta, Microsoft, Apple, Amazon)?  
 These have duplicate rates exceeding 50% for beginners → **SKIP** until you have significant experience  
  
### Activity Volume Check  
- [ ] Check 90-day report volume on the platform statistics page  
 - **15–60 reports / 90 days** on a wildcard program = healthy and competitive without being saturated → **GO**  
 - **200+ reports / 90 days** = heavy competition, high duplicate risk → **FLAG**  
 - **Under 5 reports / 90 days** = dormant or scope too narrow → **SKIP**  
  
### Hunter Count (if visible)  
- [ ] If the platform shows active hunter count:  
 - Under 200 active hunters → **GO**  
 - 1,000+ active hunters → **FLAG** — assess duplicate risk carefully  
  
---  
  
## Step 8 — Commit Decision  
  
Tally your results. Only proceed if **all** of the following are true:  
  
- [ ] Passed every check in Step 2 (zero NO-GOs)  
- [ ] Passed all hard thresholds in Steps 3, 4, and 5  
- [ ] Community sentiment in Step 6 is net neutral or positive  
- [ ] Program is not saturated per Step 7  
- [ ] Program age is in the 6-month to 3-year window (Step 1)  
  
If all pass → **COMMIT. Begin recon.**  
If any fail → **do not start. Return to Step 1 and evaluate the next candidate.**  
  
---  
  
## Step 9 — Exit Criteria (100-Hour Rule)  
  
Once engaged with a program, use these thresholds to decide whether to stay or cut losses.  
  
### Cut Losses and Leave If:  
- [ ] **100 hours invested with zero valid findings** — sunk-cost fallacy; move on  
- [ ] Triage team consistently misunderstands or dismisses technically valid reports after follow-up  
- [ ] Scope change removes the assets you were actively working on  
- [ ] Payout rate drops below 25% in updated platform metrics  
- [ ] Program pauses without an announced restart date  
- [ ] Three consecutive reports closed as N/A with no explanation after asking for clarification  
  
### Stay and Adjust If:  
- [ ] **Near-miss duplicates** — someone else found the same bug before you: you are looking in the right area, adjust timing and  
depth of methodology rather than leaving  
- [ ] Program adds new scope assets — treat this as a fresh start on new surface  
- [ ] Triage team engages substantively on rejected reports even when not paying — this is a quality signal worth staying for  
  
---  
  
## Notes  
  
<!-- Add candidate programs, evaluation scores, community research findings, and engagement observations here -->