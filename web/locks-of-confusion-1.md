# KPMG CTF 2026: Locks of Confusion - 1 — Web & Forensics Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Locks of Confusion - 1 |
| **Category** | Web Exploitation / Forensics / Key Recovery |
| **Platform** | KICyber CTF (KPMG CTF 2026 Campus Edition) |
| **Target URL** | `http://kicyber-7974d4932921-locks-of-confusion1-0-0.chals.io` |
| **Flag** | `KPMG_CTF{PKnI95kMvuXoqqaz_huqszzll_EK6RXa9aBQ4cmPOwQ}` |

---

## 1. Challenge Overview

The challenge presents an internal operations portal for an infrastructure migration. The prompt reads:

> *"Internal Ops just migrated an old database replica onto new hardware and swears everything was done 'by the book'. Somewhere sits a key that was decided long before it was ever generated."*

The clues highlight a retired legacy database replica, an audit/migration trail, and an old SSH keypair that was carried forward from an earlier system.

---

## 2. Reconnaissance & Hidden Archive Discovery

Enumerating operational routes:
```text
/inventory, /monitoring, /status, /tickets, /backups, /dns, /network, /docs, /wiki/notes
```

- In `/inventory`, a retired host was cataloged:
  ```text
  legacy-03
  Debian 4.0 (etch)
  legacy DB read-replica (decommissioned -- retained for archival/compliance)
  ```
- In `/tickets`, ticket `TICK-4471` contained hidden HTML comments:
  ```html
  <!-- TODO(infra): the tape-restore bundle from this incident never got
       cleaned up -- still sitting at
       /archive/provisioning/legacy-03-2007.tar.gz -->

  <!-- TODO(infra): same for the old keyspace mirror -- clean this up too
       once the audit fully closes: /archive/keyspace/legacy-keys-2007.tar.gz -->
  ```

Requesting these archives initially returned `403 Forbidden -- complete the access check at /verify first.`

---

## 3. Passing the `/verify` Check & Archive Extraction

1. Navigated to `/verify`, which presented an interactive coordinate verification map displaying network nodes. Clicking the requested target node dynamically unlocked download access for the session.
2. Downloaded the two archives:
   - `/archive/provisioning/legacy-03-2007.tar.gz`
   - `/archive/keyspace/legacy-keys-2007.tar.gz`

Inspecting `legacy-03-2007/var/log/provision.log`:
```text
Nov 14 2007 09:19:12 legacy-03 provision: ssh-keygen -t rsa -b 2048 -f /home/svc-legacydb/.ssh/id_rsa -N ''
Nov 14 2007 09:19:13 legacy-03 provision: public key installed to authorized_keys for svc-legacydb
```

The corresponding public key was saved in `legacy-03-2007/home/svc-legacydb/.ssh/authorized_keys`.

---

## 4. Matching the RSA Private Key

The `legacy-keys-2007.tar.gz` archive contained 300 candidate private keys (`privkey_0000.pem` to `privkey_0299.pem`).

Using Python and OpenSSL to compute the public modulus of all 300 keys and compare against the public key in `authorized_keys`:

```python
import glob, subprocess

target_pub = open("authorized_keys").read().strip().split()[1]

for key_path in sorted(glob.glob("keys/privkey_*.pem")):
    cmd = f"ssh-keygen -y -f {key_path}"
    pub = subprocess.check_output(cmd, shell=True).decode().strip().split()[1]
    if pub == target_pub:
        print("Matching key found:", key_path)
        break
```

**Match Found:** `privkey_0137.pem`.

---

## 5. Decrypting the Backup & Extracting the Flag

A `README.txt` inside the keyspace archive pointed to a final encrypted backup:
```text
/archive/backups/e6f25638ae2ed6ba955cdae6a74c6798.zip
```

The password specification:
1. Strip `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`.
2. Strip all newlines.
3. Take the first 64 characters of the raw base64 body.

Applying this rule to `privkey_0137.pem` yielded the password:
```text
MIIEowIBAAKCAQEAui2yyUzRtrahcJ1ZpfIrOBvT+s9ZI4k0mXQMegwGWfeKQ4cA
```

Decrypting `e6f25638ae2ed6ba955cdae6a74c6798.zip` unzipped `user.txt`:

```text
KPMG_CTF{PKnI95kMvuXoqqaz_huqszzll_EK6RXa9aBQ4cmPOwQ}
```

---

## 6. Final Flag

```text
KPMG_CTF{PKnI95kMvuXoqqaz_huqszzll_EK6RXa9aBQ4cmPOwQ}
```

---

## 7. Key Takeaways
- **Legacy Artifact Preservation:** Historical backups, tape restores, and provisioning bundles frequently retain credentials that remain valid across modern iterations of the infrastructure.
- **Key Pair Association:** Matching public keys in `authorized_keys` against candidate private key sets is trivial via public modulus comparison (`ssh-keygen -y`).
