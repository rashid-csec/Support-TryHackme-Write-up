# TryHackMe – Support Room Walkthrough

Room: https://tryhackme.com/r/room/support  

---

## Introduction

This walkthrough documents my penetration testing process for the TryHackMe **Support** room.

The objective was to enumerate the target system, gain initial access, escalate privileges, and retrieve both user and admin flags.

The engagement involved multiple stages including:

- Service enumeration  
- Web application testing  
- Credential attacks  
- Session manipulation  
- Command injection  

---

## 1. Network Enumeration

I started the assessment by performing a full TCP scan using Nmap to identify open ports, services, and versions running on the target system.

### Command Used

```bash
sudo nmap -sS 10.48.185.211 -T5 -A
Findings
22/tcp – OpenSSH 9.6p1 (Ubuntu)
80/tcp – Apache httpd 2.4.58 (Ubuntu)
Key Observations
Linux-based system
SSH available but no credentials initially
Web application is the main attack surface
PHP session cookies were in use
HttpOnly flag was not properly enforced

This indicated that the web application would likely be the primary entry point.

2. Web Enumeration (Gobuster)

Next, I performed directory brute-forcing using Gobuster to identify hidden files and endpoints.

Command Used
gobuster dir -u http://10.48.185.211 -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -x php,html,txt -t 50
Discovered Endpoints
/index.php
/dashboard.php
/api.php
/info.php
/config.php
/includes/
/skins/
/layout/
3. Username Discovery (Login Page Analysis)

While analyzing the login page, I identified a valid username:

help@support.thm

This email was visible within the login interface and system description.

4. Credential Attack (Hydra Brute Force)

Since no account lockout mechanism was present, I performed a brute-force attack using Hydra.

Command Used
hydra -l help@support.thm -P /usr/share/wordlists/rockyou.txt 10.48.185.211 http-post-form "/index.php:email=^USER^&password=^PASS^:F=Invalid credentials"
Result

Valid credentials were obtained:

Username: help@support.thm
Password: REDACTED
5. Initial Access & Session Analysis

After successful login, I gained access to the Support Operations Panel.

Cookies Identified
PHPSESSID
isITUser

The isITUser cookie contained a hashed value:

b326b5062b2f0e69046810717534cb09
6. Privilege Escalation via Cookie Manipulation

The application relied on insecure client-side authorization.

Exploit

The value true was hashed using MD5:

true → b326b5062b2f0e69046810717534cb09

I replaced the cookie value with this hash.

Result
Elevated privileges granted
Hidden admin features unlocked
Access control bypass achieved
7. Admin Panel Access

After modifying the cookie:

IT Admin Panel became accessible
API panel was exposed

This confirmed broken access control.

8. API Enumeration

The /api.php endpoint exposed sensitive user information.

Response Example
{
  "email": "specialadmin@support.thm",
  "2FA": false,
  "admin": true
}
Findings
Admin account exists
2FA disabled
Sensitive data exposed via API
9. Source Code Disclosure (Directory Traversal)

A directory traversal vulnerability was found in the skin parameter.

Payload
dashboard.php?skin=../config
Result

Sensitive configuration data was exposed:

$MASTER_PASSWORD = 'REDACTED';
$SITE_VER = '1.0';
$SITE_NAME = 'support portal';
10. Admin Authentication

Using the leaked credentials, I authenticated as:

specialadmin@support.thm

This granted administrative access.

11. Command Injection

A vulnerable parameter (sys) was identified that executed system commands.

Example Request
sys=date
Exploitation

Command injection was confirmed, allowing execution of system-level commands.

Impact
Remote command execution
Sensitive file access
12. Final Flag Retrieval

Using command injection, the final flag was retrieved:

/home/ubuntu/user.txt
13. Attack Chain Summary
Network enumeration (Nmap)
Web enumeration (Gobuster)
Username discovery
Credential brute force (Hydra)
Session cookie manipulation
API information disclosure
Directory traversal
Hardcoded credential exploitation
Command injection
Full system compromise
14. Conclusion

This engagement demonstrated how multiple vulnerabilities can be chained together to achieve full system compromise.

Key Security Issues
Broken access control (client-side trust)
Weak session management
Directory traversal vulnerability
Hardcoded credentials
Insecure API exposure
Command injection vulnerability

---

If you want next upgrade, I can make it:

✔ :contentReference[oaicite:0]{index=0}  
✔ :contentReference[oaicite:1]{index=1}  
✔ :contentReference[oaicite:2]{index=2}  
✔ Or :contentReference[oaicite:3]{index=3}  

Just tell me 👍
