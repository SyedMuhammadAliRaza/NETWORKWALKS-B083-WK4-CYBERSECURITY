# NETWORKWALKS-B083-WK4-CYBERSECURITY - Penetration Testing: Mediroza General Hospital
## Cybersecurity Platform Batch Status

### 📌 Overview

This repository documents Week 4 of the Cybersecurity & Ethical Hacking Internship at Networkwalks Technologies (Batch B083), focused on a full black-box penetration test against an authorized target: Mediroza General Hospital.

The engagement followed a structured four-milestone methodology - reconnaissance and initial access, encrypted file analysis, deep data exposure discovery, and professional reporting - mirroring a real-world client engagement from scoping through deliverable.

| Project Detail | Value |
|---|---|
| Client | Mediroza General Hospital |
| Target | `https://medirozahospital.com` |
| Type | Black-box Penetration Test |
| Duration | 5 Days |
| Authorization | Written permission granted by Networkwalks |

⚠️ **Disclaimer:** All activities in this repository were conducted for educational purposes within an authorized lab environment, against a target explicitly provisioned by the internship program with written authorization. No unauthorized access to third-party systems or data was performed. These techniques must never be applied to any system without explicit written permission from the owner.

### 🎯 Objectives

- Conduct structured reconnaissance against a live web target
- Identify exposed entry points and misconfigurations through passive and active enumeration
- Analyze authentication mechanisms for information disclosure
- Test for and exploit SQL injection to achieve authentication bypass
- Recover password-protected files and assess encryption strength via dictionary attacks
- Identify sensitive data exposure through server misconfiguration
- Document findings in a professional, client-ready penetration testing report

### 🧩 Milestone 1: Initial Access

**Task 1 — Reconnaissance**

Standard recon tooling was run against the target to fingerprint the hosting environment, DNS footprint, and web server stack.

```bash
whois medirozahospital.com
whatweb medirozahospital.com
nslookup medirozahospital.com
dnsrecon -d medirozahospital.com
curl -I https://medirozahospital.com
wafw00f medirozahospital.com
```

**Result:** Domain registered via Namecheap (recently created). Server fingerprinting showed inconsistent banners between tools (OpenResty vs. LiteSpeed), later attributed to a LiteSpeed WAF sitting in front of the origin, confirmed via `wafw00f`.

![whatweb mediroza](<whatweb mediroza.png>)
![whois MediRoza](<whois MediRoza.png>)
![nslookup Medirozahospital](<nslookup Medirozahospital.png>)
![curl -I Medirozahospital](<curl -I Medirozahospital.png>)
![dnsrecon Medirozahospital](<dnsrecon Medirozahospital.png>)
![wafw00f Medirozahospital](<wafw00f Medirozahospital.png>)

**Task 2 — robots.txt Disclosure**

Checking `robots.txt` revealed three disallowed directories not intended for public indexing — a strong signal for manual investigation.


![Robots Txt](<Robots Txt.png>)
![SiteMap](<SiteMap.png>)

**Task 3 — Directory Enumeration**

Browsing directly to each disclosed path revealed directory listing was enabled, exposing files not meant to be publicly browsable.

| Path | Exposure |
|---|---|
| `/patient/` | `login.php`, `portal.php`, `download.php`, `reports/` |
| `/staff/` | `login.php` |
| `/old/` | `mediroza_db_backup_2019.sql` |

![Mediroza Patient](<Mediroza Patient.png>)
![MedirozaHospital Staff](<MedirozaHospital Staff.png>)
![MediRoza Old](<MediRoza Old.png>)
![MediRoza patient records block](<MediRoza patient records block.png>)

**Task 4 — Authentication Testing**

The patient portal login form was tested for username enumeration and input handling weaknesses.

A single quote submitted in the username field returned a raw database error:


This confirmed the login form was vulnerable to **SQL injection**, with unsanitized user input passed directly into the query.

![Sql Injection](<Sql Injection.png>)
![Sql Injection 2](<Sql Injection 2.png>)

**Task 5 — Authentication Bypass**

The following payload was used in the username field to bypass authentication by commenting out the password check:


**Result:** Successful login without a valid password, granting access to the patient portal.

**Task 6 — Deliverable Retrieval**

Three confidential, password-protected pathology lab reports were retrieved from the authenticated portal:

- `patient_report_1.pdf` — Sipho Dlamini
- `patient_report_2.pdf` — Priya Reddy
- `patient_report_3.pdf` — Emily Thompson

### 🧩 Milestone 2: Data Extraction

**Task 1 — Hash Extraction**

Each encrypted PDF's password hash was extracted using `pdf2john`, producing a `$pdf$...` formatted hash compatible with John the Ripper and hashcat.

```bash
pdf2john.pl patient_report_1.pdf > report1_hash.txt
```

**Task 2 — Dictionary Attack**

A dictionary attack was run against each hash. Consistent with the milestone's guidance, a single wordlist did not succeed across all three files escalating wordlist size was required.

| File | Password | Wordlist |
|---|---|---|
| `patient_report_1.pdf` | `123456` | Top 10,000 common passwords |
| `patient_report_2.pdf` | `password` | Top 10,000 common passwords |
| `patient_report_3.pdf` | `!@#$%^&` | Resistant to dictionary attack — special-character password |

**Result:** All three reports successfully decrypted and verified by opening the unlocked copies.

### 🧩 Milestone 3: Deep Reconnaissance

**Task 1 — Revisiting Recon Findings**

The `/old/` directory disclosed in `robots.txt` during Milestone 1 was investigated directly, since it had not yet been explored.

**Task 2 — Exploiting Directory Listing**


Directory listing was enabled, exposing a legacy database backup file with no access control:


```bash
wget https://medirozahospital.com/old/mediroza_db_backup_2019.sql
```

![MediRoza old sql 1](<MediRoza old sql 1.png>)
![MediRoza old sql 2](<MediRoza old sql 2.png>)

**Task 3 — Analyzing the Exposed Backup**

The backup, generated by the site's CMS backup module in 2019, contained two sensitive tables in plaintext:

- **`staff`** - 30 records including full name, job title, department, email, phone, **national ID number**, and monthly salary
- **`shareholders`** - 10 records including shareholder name, share percentage, shares held, and share class

**Result:** Full staff compensation data and hospital ownership structure recovered from a single unauthenticated, publicly accessible file.

### 🔍 Security Observations

| Observation | Insight |
|---|---|
| Unsanitized input in authentication logic enables full bypass | A single unescaped quote in a login field compromised the entire access control layer |
| Verbose error messages leak backend implementation details | Raw `mysqli_query()` errors exposed database engine and query structure to an unauthenticated user |
| Legacy files are a persistent risk even after migration | A 2019 backup left in a `/old/` directory remained exposed years later with zero access control |
| Directory listing misconfigurations compound other findings | A single disabled index page turned a forgotten backup into a full data breach |
| Password strength varies significantly across files from the same source | Two of three PDFs used top-100-list passwords; the third required escalation to a larger, more targeted wordlist |

### 📋 Verification Checklist

- [x] Reconnaissance completed (whois, whatweb, nslookup, dnsrecon, curl, wafw00f)
- [x] robots.txt reviewed for disclosed paths
- [x] Directory listing exposure identified and exploited
- [x] Authentication mechanism tested for information disclosure
- [x] SQL injection vulnerability confirmed
- [x] Authentication bypass achieved
- [x] Three confidential PDF reports retrieved
- [x] PDF encryption hashes extracted
- [x] All three PDF passwords recovered via dictionary attack
- [x] Critical data exposure (salaries, shareholder details) identified and documented

### 🐞 Troubleshooting Log

| Issue | Cause | Resolution |
|---|---|---|
| PDF hash extraction failed via Johnny's built-in converter | `pdf2john` script missing from the installed John the Ripper build | Extracted the hash using an alternative method and verified consistency before cracking |
| Small default wordlist failed to crack one PDF | Password was not among the most common leaked passwords | Escalated to a significantly larger wordlist, consistent with real-world cracking methodology |
| Password confirmed manually but not flagged by automated tool | Mismatch traced to the cracking tool's hash extraction, not the password itself | Re-verified the correct password directly against the file before proceeding |


### 🚨 Findings Summary

| # | Vulnerability | Location | Risk Rating |
|---|---|---|---|
| 1 | Verbose database error disclosure | `patient/login.php` | Medium |
| 2 | SQL injection — authentication bypass | `patient/login.php` | Critical |
| 3 | Directory listing enabled on multiple paths | `/patient/`, `/staff/`, `/old/` | Medium |
| 4 | Confidential patient PDF reports accessible post-bypass | `patient/reports/` | High |
| 5 | Weak passwords on encrypted PDF files | `patient_report_1.pdf`, `patient_report_2.pdf` | High |
| 6 | Legacy database backup publicly exposed, unauthenticated | `/old/mediroza_db_backup_2019.sql` | Critical |
| 7 | Staff salaries and national ID numbers stored and exposed in plaintext | `mediroza_db_backup_2019.sql` | Critical |
| 8 | Shareholder ownership and equity data exposed in plaintext | `mediroza_db_backup_2019.sql` | Critical |

**Finding 1 — Verbose Database Error Disclosure (Medium)**
Submitting a single quote (`'`) in the login form's username field returned a raw `mysqli_query()` error, revealing backend database engine details and confirming unsanitized input handling to an unauthenticated user.

**Finding 2 — SQL Injection Authentication Bypass (Critical)**
The patient login form failed to sanitize or parameterize user input. The payload `admin' --` closed the SQL string early and commented out the password check entirely, granting full authenticated access without valid credentials.

**Finding 3 — Directory Listing Enabled (Medium)**
Multiple directories disclosed via `robots.txt` (`/patient/`, `/staff/`, `/old/`) returned full directory index listings instead of a 403/404, exposing internal file structure and files never intended for public access.

**Finding 4 — Confidential Reports Accessible (High)**
Once authenticated via the SQL injection bypass, three confidential pathology lab reports belonging to named patients were directly downloadable with no further access control.

**Finding 5 — Weak PDF Passwords (High)**
Two of three retrieved PDFs used passwords present in the top-100 most common leaked passwords (`123456`, `password`), cracked almost instantly via dictionary attack. The third resisted standard wordlists, indicating inconsistent password policy enforcement across files from the same system.

**Finding 6 — Exposed Legacy Database Backup (Critical)**
A 2019 database backup (`mediroza_db_backup_2019.sql`) was left in a publicly listed `/old/` directory with no authentication required, directly downloadable by anyone aware of the path.

**Finding 7 — Staff PII and Salary Disclosure (Critical)**
The exposed backup contained plaintext records for 30 staff members, including full name, job title, department, email, phone number, **national ID number**, and monthly salary — a combination sufficient for identity theft as well as internal compensation disclosure.

**Finding 8 — Shareholder Data Disclosure (Critical)**
The same backup exposed hospital ownership structure: 10 shareholders with exact share percentages, share counts, and share class - commercially sensitive information that could materially affect the organization if disclosed publicly or to competitors.

### ✅ Conclusion

This engagement demonstrated that Mediroza General Hospital's web infrastructure contains multiple critical vulnerabilities that, when chained together, result in a complete compromise of patient confidentiality, employee privacy, and corporate ownership data.

The root cause was not a single catastrophic flaw but a combination of basic, well-understood weaknesses: unsanitized database queries, misconfigured directory indexing, and inadequate password policy enforcement. Individually, several of these findings might be rated Medium; in combination, they formed a clear attack path — from an unauthenticated login form, through full authentication bypass, to the retrieval of regulated patient health data and, ultimately, to a completely unrelated and more severe exposure of staff and shareholder records via a forgotten legacy backup file.

Of particular concern is that the most damaging finding (Finding 6–8) did not require defeating any security control at all - it required only navigating to a URL disclosed in the site's own `robots.txt` file. This underscores a critical lesson for the client: security controls that rely on obscurity (an undocumented path, a file "nobody would think to check") provide no real protection once any part of the attack surface is enumerated.

Given the healthcare context of this target, the exposure of patient lab results alongside staff national ID numbers and salary data would, in a real-world scenario, likely constitute a reportable data breach under applicable data protection regulations. Remediation should be treated as urgent, prioritizing the Critical-rated findings (SQL injection, directory listing, and backup file exposure) before any Medium or High findings.

Overall, this engagement successfully achieved all four milestone objectives, demonstrating a complete, evidence-backed attack chain suitable for presentation to the client as a professional penetration testing report.

### 💡 What I Learned

This engagement reinforced that the most severe vulnerabilities often come from layered, compounding weaknesses rather than a single flaw. The SQL injection in the login form alone would have been serious, but it was the chain — injection → authentication bypass → file retrieval — that demonstrated real, provable impact to a client.

The `/old/` directory exposure was a strong reminder that reconnaissance findings should never be treated as one-time checks. A path disclosed early in the engagement (via `robots.txt`) became the single most critical finding of the entire test once properly investigated. It also highlighted a common real-world failure mode: legacy backups left behind after a migration, forgotten but still reachable, with no authentication and no expiry.

On the cracking side, the exercise confirmed that encryption strength and password strength are independent variables — all three files used identical encryption, yet required three different cracking strategies based on password complexity alone.

### 🔗 Resources

- [OWASP — SQL Injection](https://owasp.org/www-community/attacks/SQL_Injection)
- [OWASP — Testing for Directory Traversal / Listing](https://owasp.org/www-project-web-security-testing-guide/)
- Openwall — John the Ripper
- 

**Author**
Syed Muhammad Ali Raza
Cybersecurity & Ethical Hacking Intern — Networkwalks Technologies (Batch B083)
