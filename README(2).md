# Web Application Attacks Lab — OWASP Juice Shop + Burp Suite

## 1. Project Title

**Web Application Attacks Lab: OWASP Juice Shop | Burp Suite, Docker, OWASP Top 10**

## 2. Overview

This is the third lab in a series of hands-on cybersecurity exercises completed as part of a BSc IT: Security & Network Engineering programme. While Labs 1 and 2 focused on network and host-level attacks, this lab shifts to **web application security testing** — the most common real-world attack surface, since nearly every organisation exposes a website.

The target application is **OWASP Juice Shop**, an intentionally vulnerable web application maintained by the Open Worldwide Application Security Project (OWASP) specifically for security training. Juice Shop was deployed locally in a Docker container on a Kali Linux VM and tested using **Burp Suite Community Edition** as an intercepting proxy.

## 3. Objectives

- Deploy a deliberately vulnerable web application in an isolated, local environment.
- Configure and use Burp Suite to intercept, inspect, and modify live HTTP traffic.
- Identify and demonstrate common web application vulnerabilities through hands-on exploitation.
- Document each finding with evidence, root cause, impact, and remediation.
- Practise mapping findings to recognised web application vulnerability categories.

## 4. Lab Environment / Tools

| Component | Details |
|---|---|
| Host OS | Kali Linux (VM, reused from Labs 1 and 2) |
| Target Application | OWASP Juice Shop (`bkimminich/juice-shop`) |
| Deployment Method | Docker |
| Proxy Tool | Burp Suite Community Edition |
| Browser | Burp Suite's built-in browser |
| Target Address | `http://127.0.0.1:3000` / `http://localhost:3000` (local only) |

## 5. Lab Architecture / Setup

Juice Shop was pulled and run as a Docker container on the Kali VM, bound to local port 3000:

```bash
sudo apt update
sudo apt install docker.io -y
sudo systemctl enable --now docker

sudo docker --version
sudo docker run hello-world

sudo docker pull bkimminich/juice-shop
sudo docker run -d -p 3000:3000 bkimminich/juice-shop
```

Docker commands required `sudo` throughout. After the container started, Juice Shop was confirmed reachable at `http://localhost:3000`.

*Evidence: `W1-docker-hello-world.png`, `W2-juice-shop.png`*

Burp Suite was then configured as the intercepting proxy (listener on `127.0.0.1:8080`). After initial difficulty routing external Firefox traffic through Burp, Burp Suite's **built-in browser** was used instead, which successfully routed all Juice Shop traffic through the proxy for interception and analysis.

*Evidence: `W3-burp-suite.png`, `W4-burp-intercept.png`*

All testing described below was performed exclusively against this local, self-hosted instance of Juice Shop. No external or third-party system was targeted at any point.

## 6. Methodology

1. Deploy the vulnerable application in an isolated local environment.
2. Configure an intercepting proxy to observe and manipulate traffic between the browser and the application.
3. Test each target area of the application against a known vulnerability class.
4. Capture evidence of the request, the response, and the resulting impact.
5. Map each finding to its underlying vulnerability category and document remediation.

---

## 7. Finding 1 — SQL Injection (Authentication Bypass)

**What it is:**
SQL Injection occurs when untrusted user input is concatenated directly into a database query without proper validation or parameterisation, allowing an attacker to alter the query's logic.

**What I did:**
On the Juice Shop login page, the following payload was entered into the email field, with an arbitrary value in the password field:

```
' OR 1=1--
```

**Why it works:**
- The closing single quote (`'`) terminates the original email string in the underlying SQL query.
- `OR 1=1` introduces a condition that always evaluates to true, matching every row in the users table.
- `--` comments out the remainder of the original query, including the password check.

**What the evidence showed:**
The login was successfully bypassed without knowledge of a valid password, logging in as an existing account. Burp Suite's Intercept feature was then used to capture the raw `POST` request to `/rest/user/login`, confirming the injection payload was sent verbatim in the request body before being forwarded to the server.

*Evidence: `W5-sqli-login-bypass.png` (payload entry and successful bypass), `W6-sqli-burp-request.png` (raw request body captured in Burp)*

**Security impact:**
Authentication could be bypassed entirely, allowing unauthorised access to an account without valid credentials. In a production system, this could expose personal data, order history, or administrative functionality.

**Remediation:**
- Use parameterised queries / prepared statements for all database access.
- Never concatenate untrusted user input directly into SQL statements.
- Apply consistent server-side input validation.

---

## 8. Finding 2 — Cross-Site Scripting (XSS)

**What it is:**
XSS occurs when an application renders untrusted input in a way that causes a browser to execute it as code rather than display it as plain text, allowing an attacker-controlled script to run in a user's browser session.

**What I did:**
The Lab 3 tutorial's intended payload, `<script>alert('XSS')</script>`, was first tested against the Juice Shop search field but did not execute against the version of Juice Shop used in this lab. Because modern Juice Shop builds filter the `<script>` tag directly, the following DOM XSS payload was submitted through the **search box** instead:

```
<iframe src="javascript:alert(`xss`)">
```

**What the evidence showed:**
The payload executed successfully, producing a JavaScript `alert()` pop-up reading `xss`, confirming the browser interpreted the submitted input as executable code rather than plain search text. The screenshot shows the payload reflected in the search URL and the resulting alert box.

*Evidence: `W7-xss-alert.png`*

**Note on scope of finding:** The Lab 3 tutorial frames this exercise under "Stored XSS." The evidence gathered in this lab demonstrates that JavaScript executed via the Juice Shop **search functionality** — it does not independently confirm that the payload was persisted server-side and re-triggered for other users. This finding is therefore reported as **confirmed XSS execution via the search feature**, rather than a confirmed stored/persistent XSS, to keep the write-up accurate to the evidence collected.

**Security impact:**
Successful script execution in this context demonstrates that user-supplied input is not being safely handled before being rendered. In a real-world deployment this could be leveraged to steal session cookies, hijack sessions, or redirect users to malicious sites.

**Remediation:**
- Apply context-aware output encoding wherever user input is rendered.
- Sanitise and validate input on both client and server.
- Avoid inserting untrusted input into executable HTML/JavaScript contexts.

---

## 9. Finding 3 — Broken Access Control (IDOR)

**What it is:**
Insecure Direct Object Reference (IDOR) occurs when an application exposes an internal object (such as a database record) via a user-controllable identifier (such as an ID in a URL) without verifying that the requesting user is actually authorised to access that object.

**What I did:**
While logged into my test account, I added a product to my shopping basket. Using Burp Suite's Intercept feature, I captured the resulting request:

```
GET /rest/basket/1
```

This confirmed my own basket's object ID was `1`. The request was sent to **Burp Repeater**, where only the basket ID was changed:

```
GET /rest/basket/2
```

**What the evidence showed:**
The server responded `HTTP/1.1 200 OK` with a JSON body reporting `"status": "success"` and returned basket data for `"id": 2`, `"UserId": 2`, including product details (e.g. a Raspberry Juice item) belonging to a different basket than my own.

*Evidence: `W8-idor-burp-repeater.png`*

**Security impact:**
The server returned another user's basket data based solely on a modified numeric ID, with no server-side check confirming the requesting account owned that basket. In a real deployment, this could allow systematic enumeration of basket or order IDs to access other customers' data.

**Remediation:**
- Perform server-side authorisation checks on every object request.
- Verify that the requested resource belongs to the authenticated user before returning data.
- Never rely on the obscurity of an object ID as an access control mechanism.

> **Scope note:** This test was performed only against my own local Juice Shop test accounts. No real users or production systems were involved.

---

## 10. Finding 4 — Sensitive Data Exposure

**What it is:**
Sensitive data exposure occurs when files or directories that were never meant to be publicly accessible — such as backups, configuration files, or internal documents — are left reachable by any visitor, requiring no exploit beyond simply knowing (or guessing) the path.

**What I did:**
I navigated directly to:

```
http://localhost:3000/ftp
```

**What the evidence showed:**
The directory was publicly browsable and listed numerous files, including:

- `coupons_2013.md.bak`
- `incident-support.kdbx`
- `package.json.bak`
- `acquisitions.md`
- `eastere.gg`
- `legal.md`
- `suspicious_errors.yml`
- `announcement_encrypted.md`
- `encrypt.pyc`
- `package-lock.json.bak`
- `quarantine/` (directory)

*Evidence: `W9-exposed-ftp-directory.png`*

I then opened `http://127.0.0.1:3000/ftp/acquisitions.md`, a document titled **"Planned Acquisitions"** and marked *"This document is confidential! Do not distribute!"*, containing fictional details about planned company acquisitions and potential stock market impact.

*Evidence: `W10-exposed-file.png`*

**Security impact:**
Sensitive internal documents and backup files were exposed with no authentication or access control, demonstrating how confidential information can leak through simple misconfiguration rather than a sophisticated exploit.

**Remediation:**
- Remove backup and unnecessary files from production web directories before deployment.
- Disable directory listing on web servers.
- Restrict access to sensitive files through proper access controls.
- Store confidential documents outside of publicly served directories.

> **Note:** All content in `/ftp` is part of the intentionally vulnerable Juice Shop training application. No real company's confidential data was accessed at any point.

---

## 11. Evidence / Screenshots

| File | Phase | Description |
|---|---|---|
| `W1-docker-hello-world.png` | 1 | Docker installed; `hello-world` container runs successfully |
| `W2-juice-shop.png` | 1 | Juice Shop homepage loaded at `localhost:3000` |
| `W3-burp-suite.png` | 2 | Burp Suite main window after launch |
| `W4-burp-intercept.png` | 2 | Burp Proxy Intercept pausing a raw HTTP request |
| `W5-sqli-login-bypass.png` | 3 | SQL injection payload entered; successful authentication bypass |
| `W6-sqli-burp-request.png` | 3 | Raw `POST /rest/user/login` request captured in Burp, showing the SQLi payload |
| `W7-xss-alert.png` | 4 | JavaScript `alert()` triggered via the search field, confirming XSS execution |
| `W8-idor-burp-repeater.png` | 5 | Burp Repeater: modified basket ID request and the resulting unauthorised basket data |
| `W9-exposed-ftp-directory.png` | 6 | Publicly browsable `/ftp` directory listing |
| `W10-exposed-file.png` | 6 | Contents of an exposed file (`acquisitions.md`) from `/ftp` |

## 12. Key Lessons Learned

- **SQL Injection and XSS** both stem from unsafe handling of user-supplied input being interpreted as code (SQL or JavaScript) rather than data.
- **IDOR** is a different category of flaw entirely — the input handling is fine, but the server fails to verify that a requester is authorised to access the specific object they're asking for.
- **Sensitive data exposure** requires no exploit technique at all; it results from operational oversight — files that should never have been deployed to a public-facing directory.
- Burp Suite proved essential throughout the lab for:
  - Intercepting and pausing live HTTP requests
  - Inspecting raw request bodies before they reached the server
  - Modifying request parameters (e.g. object IDs)
  - Sending requests to **Repeater** for controlled, repeatable testing
  - Comparing request/response pairs to confirm a vulnerability

## 13. Remediation Summary

| Finding | Root Cause | Recommended Fix |
|---|---|---|
| SQL Injection | Unsafe SQL query construction using unvalidated input | Parameterised queries / prepared statements |
| XSS | Unsafe handling/rendering of user-controlled input | Context-aware output encoding, sanitisation, and safe rendering |
| IDOR | Missing server-side ownership/authorisation checks | Enforce authorisation checks on every object request |
| Sensitive Data Exposure | Sensitive/backup files left publicly accessible | Remove exposed files, restrict access, disable unnecessary directory listing |

## 14. Ethical / Legal Scope

- All testing in this project was performed exclusively against **OWASP Juice Shop**, an application built specifically for security training.
- Juice Shop was run locally, in a Docker container, on my own Kali Linux VM — it was never exposed to the internet or any external network.
- This exercise was conducted for **authorised, educational purposes only**, as part of a university cybersecurity course.
- No real company's website, infrastructure, or data was targeted at any point.
- No real customer or user data was accessed.
- The techniques demonstrated here (SQL injection, XSS, IDOR testing, directory enumeration) should **never** be used against any system without explicit, written authorisation. Doing so is illegal.

## 15. Conclusion

This lab provided hands-on experience across four distinct categories of web application vulnerability — injection, cross-site scripting, broken access control, and sensitive data exposure — using an industry-standard interception proxy against a purpose-built vulnerable application. It reinforced that web application security requires attention to multiple, often unrelated failure modes: unsafe input handling, missing authorisation logic, and basic operational hygiene around what gets deployed publicly.

---

## Skills Demonstrated

- Web application security testing
- Burp Suite (Proxy, Intercept, Repeater)
- HTTP request interception and manipulation
- SQL Injection testing
- Cross-Site Scripting (XSS) testing
- Authentication bypass testing
- Broken access control / IDOR testing
- Sensitive data exposure assessment
- Docker container deployment
- Kali Linux
- OWASP Juice Shop
- Vulnerability documentation and reporting
- Security remediation analysis
