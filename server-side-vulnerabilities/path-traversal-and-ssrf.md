# PortSwigger Lab Write-Up: Path Traversal & Server-Side Request Forgery (SSRF)

## Overview
This repository documents practical walkthroughs and remediation strategies for server-side vulnerabilities involving arbitrary file access and unintended backend server requests, completed on the PortSwigger Web Security Academy platform.

---

## Part 1: Path Traversal (Directory Traversal)

Path traversal allows an attacker to read arbitrary files on the server running an application. This may include application source code, configuration files, backend credentials, or sensitive operating system files like `/etc/passwd`.

### 1. Lab: Traversal Sequences Blocked with Absolute Path Bypass
* **Vulnerability:** The application blocks relative traversal sequences (`../`), but accepts absolute file paths directly.
* **Exploitation:**
  1. Intercept the product image request using Burp Suite Proxy.
  2. Change the filename parameter to reference the absolute system path:
     ```http
     GET /image?filename=/etc/passwd HTTP/1.1
     ```
  3. The server retrieves and displays `/etc/passwd`.

### 2. Lab: Traversal Sequences Stripped Non-Recursively
* **Vulnerability:** The application uses a naive filter that removes standard `../` sequences once without stripping them recursively.
* **Exploitation:**
  1. Intercept the image request in Burp Suite.
  2. Use nested traversal sequences (`....//` or `....\/`):
     ```http
     GET /image?filename=....//....//....//etc/passwd HTTP/1.1
     ```
  3. When the inner `../` is stripped, the remaining characters collapse back into `../`, successfully traversing directories.

### 3. Lab: Traversal Sequences Stripped with Superfluous URL-Decode
* **Vulnerability:** Input filters execute before an unneeded second URL-decode step on the server.
* **Exploitation:**
  1. Intercept the request loading the product image.
  2. Double URL-encode the traversal payload (`.` -> `%252e`, `/` -> `%252f`):
     ```http
     GET /image?filename=..%252F..%252F..%252Fetc%252Fpasswd HTTP/1.1
     ```
  3. The filter inspects `%252F` (which is harmless) before the backend decodes it to `/`.

### 4. Lab: Validation of Start of Path
* **Vulnerability:** The application validates that the input starts with the expected base directory (e.g., `/var/www/images`).
* **Exploitation:**
  1. Provide the expected directory path followed immediately by traversal sequences:
     ```http
     GET /image?filename=/var/www/images/../../../etc/passwd HTTP/1.1
     ```
  2. The prefix check passes, and the path traverses back to the root directory.

### 5. Lab: Validation of File Extension with Null Byte Bypass
* **Vulnerability:** The server requires a specific extension (e.g., `.jpg`, `.png`), but backend filesystem APIs in older frameworks/languages terminate string processing upon encountering a null byte (`%00`).
* **Exploitation:**
  1. Append a null byte followed by the accepted extension:
     ```http
     GET /image?filename=../../../etc/passwd%00.jpg HTTP/1.1
     ```
  2. The validation check sees `.jpg`, but the filesystem stops reading at `%00` and opens `/etc/passwd`.

### Path Traversal Remediation
* Avoid passing user-controlled input directly to filesystem APIs.
* Maintain a hardcoded whitelist of allowed file names.
* If user input is required:
  1. Restrict input strictly to alphanumeric characters.
  2. Canonicalize the combined file path using platform APIs (e.g., `getCanonicalPath()` in Java or `realpath()` in PHP).
  3. Verify that the canonical path starts with the designated base directory.

---

## Part 2: Server-Side Request Forgery (SSRF)

SSRF occurs when a server-side application makes HTTP requests to an arbitrary domain or internal system specified by user input.

### 1. Lab: Basic SSRF Against Localhost & Admin Functionality
* **Vulnerability:** A stock-checking feature forwards a backend API URL specified inside a `POST` parameter without adequate validation.
* **Exploitation:**
  1. Intercept the "Check Stock" POST request in Burp Suite Proxy.
  2. Modify the `stockApi` parameter to point to `http://localhost/admin`.
  3. Note that the administrative endpoint exposes an action to delete a user (`/admin/delete?username=carlos`).
  4. URL-encode the destination URL inside the POST body to complete the action:
     ```http
     POST /product/stock HTTP/1.1
     Host: vulnerable-website.com
     Content-Type: application/x-www-form-urlencoded

     stockApi=http%3A%2F%2Flocalhost%2Fadmin%2Fdelete%3Fusername%3Dcarlos
     ```

### 2. Lab: SSRF Against Another Backend System (Internal Network Scanning)
* **Vulnerability:** The web server can reach internal network devices in the private `192.168.0.0/24` subnet on port `8080`.
* **Exploitation:**
  1. Intercept the check-stock request in Burp Suite and send it to **Burp Intruder**.
  2. Set a payload position on the last octet of the IP:
     ```http
     stockApi=http://192.168.0.§1§:8080/admin
     ```
  3. Run a number payload scan from `1` to `255`.
  4. Identify IP `192.168.0.203` returning an HTTP `200 OK` response.
  5. Issue the final deletion request to solve the lab:
     ```http
     stockApi=http://192.168.0.203:8080/admin/delete?username=carlos
     ```

### SSRF Remediation
* Enforce strict input validation using an allowlist of permitted internal endpoints and protocols.
* Disable HTTP redirection following in backend HTTP client configurations.
* Segment networks and implement firewall rules preventing the web tier from accessing internal management interfaces.
