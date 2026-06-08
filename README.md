# 🔐 TryHackMe – Support Room Walkthrough

<p align="center">
  <img width="745" height="162" alt="Screenshot 2026-06-08 120748" src="https://github.com/user-attachments/assets/def35db3-e224-4090-9c60-ddd5988251b6" />


## 📖 Introduction

In this walkthrough, I document my approach to solving the **Support** room on TryHackMe. The objective was to enumerate the target, identify vulnerabilities within the web application, gain administrative access, and ultimately retrieve the available flags.

Throughout the assessment, I encountered several vulnerabilities that could be chained together to achieve full compromise of the application. These included weak authentication controls, insecure session management, information disclosure, directory traversal, and command injection.

> **Note:** This walkthrough was completed in a controlled TryHackMe lab environment for educational purposes.

---

# 🔍 Step 1: Network Enumeration

As with any penetration test, I began by identifying exposed services on the target system.

## Command

```bash
sudo nmap -sS <TARGET-IP> -T5 -A
```

## Results

The scan revealed the following services:

| Port | Service | Version                |
| ---- | ------- | ---------------------- |
| 22   | SSH     | OpenSSH 9.6p1 (Ubuntu) |
| 80   | HTTP    | Apache 2.4.58 (Ubuntu) |

The web service immediately stood out as the most promising attack surface.
<p align="center">
<img width="1908" height="768" alt="nmap" src="https://github.com/user-attachments/assets/effd3f81-a189-4e78-8ac1-301b0343f088" />


### Notes

* The target appeared to be running Linux.
* SSH was accessible but I did not yet have credentials.
* The HTTP service hosted a web application named **Support Operations Panel**.
* Session cookies were already visible in the browser.

At this stage, I decided to focus entirely on the web application.

---

# 🌐 Step 2: Web Enumeration

To understand the application's structure, I performed directory enumeration.

## Command

```bash
gobuster dir -u http://<TARGET-IP> \
-w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
-x php,html,txt -t 50
```

## Results

The scan revealed several interesting endpoints:

```text
/index.php
/dashboard.php
/api.php
/info.php
/config.php
/includes/
/skins/
/layout/
```

<p align="center">
<img width="1915" height="684" alt="gobustr" src="https://github.com/user-attachments/assets/e364fdb9-7447-4143-9b59-9774a15b9177" />

### Notes

The presence of files such as `config.php` and `api.php` suggested that sensitive functionality might be exposed through the application.

I made a note of these files for further investigation later in the assessment.

---

# 👤 Step 3: Username Discovery

Before attempting any password attacks, I looked for valid usernames.

While reviewing the login page and application content, I discovered the following email address:

```text
help@support.thm
```
<p align="center">
<img width="1919" height="885" alt="login" src="https://github.com/user-attachments/assets/d3530a13-4d82-4e86-8ebf-d31675b0093f" />


### Notes

This appeared to be a legitimate user account rather than placeholder content.

Since valid usernames are often the hardest part of authentication attacks, this immediately became a high-value finding.

---

# 🔑 Step 4: Password Brute Force

I noticed that the application did not appear to implement rate limiting or account lockout protection.

To test this, I launched a password brute-force attack using Hydra.

## Command

```bash
hydra -l help@support.thm \
-P /usr/share/wordlists/rockyou.txt \
<TARGET-IP> http-post-form \
"/index.php:email=^USER^&password=^PASS^:F=Invalid credentials"
```

## Result

The attack successfully identified valid credentials for the user account.

<p align="center">
<img width="1919" height="562" alt="hydra" src="https://github.com/user-attachments/assets/7534d1e3-f9aa-4411-a13e-606fbef9d896" />


### Notes

The absence of brute-force protection significantly weakened the authentication mechanism.

After obtaining valid credentials, I authenticated to the application and began reviewing available functionality.

---

# 🍪 Step 5: Session Analysis

Once logged in, I inspected the session cookies using browser developer tools.

## Cookies Observed

```text
PHPSESSID
isITUser
```

The `PHPSESSID` cookie appeared to be a standard session identifier.

The `isITUser` cookie was more interesting because it appeared to control user permissions.

<p align="center">
<img width="1919" height="625" alt="crackstation" src="https://github.com/user-attachments/assets/c867abee-3afa-4a24-948f-a8ffda506c1f" />


### Notes

Whenever I encounter application-specific cookies, I always test whether they influence authorization decisions.

This cookie became the focus of my next stage of testing.

---

# ⬆️ Step 6: Privilege Escalation Through Cookie Manipulation

I began modifying the `isITUser` cookie to observe how the application responded.

After changing the cookie value and refreshing the application, additional functionality became available.

### Result

I gained access to:

* IT Admin Panel
* API Viewer
  
<p align="center">
<img width="1916" height="555" alt="admin-panel" src="https://github.com/user-attachments/assets/3893a4a8-aa55-4b41-b8ac-bd6d19b98a7a" />


### Notes

This confirmed that authorization decisions were being made on the client side rather than the server side.

This is a classic example of **Broken Access Control**, where users can elevate privileges simply by modifying browser-controlled values.

---

# 🛠️ Step 7: Administrative Functionality

With elevated privileges, I explored the newly accessible administrative features.

Several administrative functions became visible that were previously hidden.

<p align="center">
<img width="1919" height="862" alt="api" src="https://github.com/user-attachments/assets/22dacfea-a69e-413a-b827-6a840eff11bb" />


### Notes

Administrative panels often expose additional attack surfaces, so I began reviewing every new page and endpoint for weaknesses.

---

# 📡 Step 8: API Enumeration

One of the newly accessible features was an API endpoint.

The API returned internal user information, including administrative account details.

## Example Response

```json
{
  "email": "specialadmin@support.thm",
  "2FA": false,
  "admin": true
}
```

<p align="center">
<img width="1021" height="486" alt="specialadmin" src="https://github.com/user-attachments/assets/b9921400-8c96-4bc5-8fbb-0f8c0d869c72" />


### Notes

This response revealed three important pieces of information:

1. An administrative account existed.
2. Two-factor authentication was disabled.
3. The account possessed administrative privileges.

This information became useful during the next phase of exploitation.

---

# 📂 Step 9: Source Code Disclosure

While reviewing application parameters, I discovered that the `skin` parameter accepted user-controlled input.

I tested for directory traversal using the following payload:

```text
dashboard.php?skin=../config
```

<p align="center">
<img width="1246" height="484" alt="sp-pass" src="https://github.com/user-attachments/assets/f1cd7b8d-5483-44d0-b770-e61edbde3632" />


## Result

The application disclosed backend configuration information.

### Notes

Configuration files often contain credentials, API keys, or sensitive internal information.

The exposed configuration revealed a hardcoded password value, which became my next target.

---

# 🔐 Step 10: Administrative Authentication

Using information gathered from the exposed configuration, I attempted to authenticate as the administrative user.

After testing the disclosed credentials and minor variations, I successfully authenticated as:

```text
specialadmin@support.thm
```

### Result

Administrative access was obtained.

<p align="center">
<img width="1919" height="849" alt="sp-flg" src="https://github.com/user-attachments/assets/9f8d04c7-e6c0-4a25-9c7b-f483eba6238e" />


### Notes

Hardcoded credentials represent a significant security risk because compromise of source code or configuration files immediately exposes privileged access.

---

# 💥 Step 11: Command Injection

While analyzing requests in Burp Suite, I identified a parameter named `sys`.

The application appeared to pass this value directly to underlying system commands.

## Example Request

```http
POST /dashboard.php?skin=green

sys=date
```

I tested various command injection techniques and confirmed that arbitrary command execution was possible.

<p align="center">
<img width="1919" height="832" alt="sys" src="https://github.com/user-attachments/assets/5ff43482-b063-4d9e-921d-f95a43eb1182" />


### Notes

This was the most critical vulnerability discovered during the assessment because it allowed direct interaction with the operating system.

---

# 🏁 Step 12: Final Flag Retrieval

Using the command injection vulnerability, I accessed sensitive files on the target system.

The final flag was successfully retrieved from:

```text
/home/ubuntu/user.txt
```
<p align="center">
<img width="1919" height="837" alt="flag" src="https://github.com/user-attachments/assets/05e06109-4bdd-41be-8206-f63e040a6625" />


At this point, the objectives of the room had been completed.

---

# 🔗 Attack Path Summary

```text
Nmap Enumeration
      ↓
Web Enumeration
      ↓
Username Discovery
      ↓
Password Brute Force
      ↓
Application Access
      ↓
Cookie Manipulation
      ↓
Privilege Escalation
      ↓
API Enumeration
      ↓
Directory Traversal
      ↓
Configuration Disclosure
      ↓
Administrative Authentication
      ↓
Command Injection
      ↓
Flag Retrieval
```

---

# 📚 Lessons Learned

This room demonstrates how several seemingly minor vulnerabilities can be chained together to achieve complete compromise.

Key weaknesses identified included:

* Weak authentication controls
* Broken access control
* Information disclosure
* Directory traversal
* Hardcoded credentials
* Command injection

The exercise reinforced the importance of thorough enumeration, as each vulnerability ultimately led to the discovery of the next.

---

**Room Completed Successfully ✅**
