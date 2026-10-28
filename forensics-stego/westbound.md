# KPMG CTF 2026: Westbound — OSINT, Forensics & Git Writeup

## Challenge Overview

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Westbound |
| **Category** | OSINT / Digital Forensics / Cryptography / Git |
| **Points** | 30 |
| **Target Organization** | Kestrel Maritime |
| **Threat Actor** | `nulltide` |
| **Artifact** | `westbound.zip` |
| **Flag** | `KPMG_CTF{g1t_r3m3mb3r5_wh4t_y0u_d3l3t3}` |

---

## 1. Incident Scenario

An extortionist operating under the alias `nulltide` claimed to have exfiltrated a 41 GB confidential archive from shipping firm **Kestrel Maritime**, containing westbound manifests, SLA annexes, and sensitive port schedules. The attacker encrypted the stolen export and planted a marker to prove their access.

The provided `westbound.zip` contained an interactive investigation package including:
- An incident briefing & instructions (`README.txt`, `index.html`)
- An attacker phishing email (`phish.eml`)
- A photographic clue (`sites/files/desk.jpg`)
- A network packet capture (`sites/files/westbound.pcap`)
- An audio recording (`sites/files/call.wav`)
- A simulated dead-drop portal (`sites/vault/`)
- A public Git repository clone (`sites/git/`)

---

## 2. Tools Used

- **unzip / 7-Zip** — Archive extraction
- **Python 3** — Email parsing and artifact decoding
- **Wireshark / strings** — Network traffic inspection
- **Audacity** — Spectrogram audio analysis
- **OpenSSL** — AES-256-CBC PBKDF2 decryption
- **Git** — Git commit object history analysis

---

## 3. Step-by-Step Investigation & Solution

### Step 1: Initial Extraction & Triage
Extracted the challenge bundle and hosted the local static evidence mirror:

```bash
unzip westbound.zip -d westbound
cd westbound
cat README.txt
python3 -m http.server 8000
```

The briefing emphasized that all domains in the scenario were simulated locally and that the encryption key was not directly present in any email or web text.

---

### Step 2: Phishing Email Attachment Extraction
The email `phish.eml` came from `nulltide@nulltide.example` warning Kestrel Maritime of the encryption. Using Python's `email` module, I extracted the attachment:

```python
from email import policy
from email.parser import BytesParser

with open("phish.eml", "rb") as f:
    msg = BytesParser(policy=policy.default).parse(f)

for part in msg.walk():
    if part.get_content_disposition() == "attachment":
        filename = part.get_filename() or "attachment.bin"
        with open(filename, "wb") as out:
            out.write(part.get_payload(decode=True))
        print("Extracted:", filename)
```

The extracted file `proof.png` showed a screenshot of an exfiltration terminal. Crucially, a note indicated that the encryption key was spoken rather than written down.

---

### Step 3: Recovering the Vault Drop Code from Photography
Inspecting the attacker's blog posts revealed that the access URL for the dead-drop vault was recorded in a photograph in the trophy locker (`desk.jpg`).

Examining `desk.jpg` showed a handwritten sticky note labeled **vault drop code**:
```text
7c41f9a2e0b3
```

Following the blog's route structure, the corresponding vault endpoint was located at:
```text
sites/vault/7c41f9a2e0b3/index.html
```

Visiting this page confirmed the vault existed, but it required an **access phrase** before releasing the encrypted download.

---

### Step 4: Recovering the Access Phrase from Network Traffic
The vault instructions stated that the access phrase had previously been submitted over HTTP while an internal network sensor was monitoring.

I examined `westbound.pcap` using `strings` and `grep`:

```bash
strings westbound.pcap | grep -Ei 'POST|phrase=|vault'
```

Found the relevant HTTP request:
```http
POST /api/login HTTP/1.1
Host: vault.nulltide.example
Content-Type: application/x-www-form-urlencoded

user=nulltide&phrase=velvet-harbor
```

The access phrase was:
```text
velvet-harbor
```

Inspecting the vault page's client-side JavaScript confirmed that `SHA256("velvet-harbor")` matched the expected verification hash:
```text
251eb9febd456d3f6b3e16aff9f0c90faec5c29ac2ed01cbfd143762a11c6eed
```

Submitting `velvet-harbor` released the protected file:
```text
westbound-export.zip.enc
```

---

### Step 5: Audio Spectrogram Analysis
The downloaded archive was encrypted with OpenSSL AES-256-CBC, but the passphrase was missing. The remaining clue was `call.wav` (a recording from the company helpdesk line).

Listening to the audio played normal white noise and speech, but opening `call.wav` in **Audacity** and switching the track display from **Waveform** to **Spectrogram** revealed visual frequency text:

```text
ARCHIVE KEY s4lt3d-t1d35
```

The exact decryption passphrase was:
```text
s4lt3d-t1d35
```

---

### Step 6: Decrypting the Export Archive
The vault specification indicated OpenSSL AES-256-CBC with PBKDF2 (SHA-256, 200,000 iterations). I decrypted the file using OpenSSL:

```bash
openssl enc -d -aes-256-cbc \
  -pbkdf2 -iter 200000 -md sha256 \
  -in westbound-export.zip.enc \
  -out westbound-export.zip \
  -pass pass:'s4lt3d-t1d35'
```

Tested and unpacked the decrypted zip:

```bash
unzip -t westbound-export.zip
unzip westbound-export.zip -d recovered
cd recovered/tidewatch
```

---

### Step 7: Recovering the Deleted Flag from Git History
The decrypted directory contained a Git repository named `tidewatch`. Inspecting the commit log:

```bash
git log --oneline --all
```

Output:
```text
72eb7a7 remove notes before push
5ff35ab notes before i forget
6b26aa5 tag the fields i care about
fabe99b init: host lookup
```

Commit `5ff35ab` added a file named `notes.b64`, which was subsequently deleted in commit `72eb7a7`. Even though `notes.b64` was missing from the current working tree, Git's object database retained the commit data.

Inspected and decoded the deleted file directly:

```bash
git show 5ff35ab:notes.b64 | base64 -d
```

The output revealed the planted marker:
```text
KPMG_CTF{g1t_r3m3mb3r5_wh4t_y0u_d3l3t3}
```

---

## 4. Final Flag

```text
KPMG_CTF{g1t_r3m3mb3r5_wh4t_y0u_d3l3t3}
```

---

## 5. Summary of Investigation Evidence Chain

```
[Phish.eml] ──> Extracted proof.png (Clue: Key is spoken, not written)
      │
[desk.jpg]  ──> Sticky note reveals Vault Drop Code: "7c41f9a2e0b3"
      │
[westbound.pcap] ──> HTTP POST traffic reveals Access Phrase: "velvet-harbor"
      │
[Vault Portal] ──> Unlocks download: westbound-export.zip.enc
      │
[call.wav]  ──> Spectrogram frequency visual shows Key: "s4lt3d-t1d35"
      │
[OpenSSL]   ──> Decrypts AES-256-CBC PBKDF2 archive to tidewatch Git repo
      │
[Git Logs]  ──> `git show 5ff35ab:notes.b64 | base64 -d` yields Flag!
```

---

## 6. Key Takeaways
- **Git preserves everything:** Deleting a file from the working directory or staging area does not purge it from Git history. Always use `git log --all -- <filename>` or `git filter-repo` to audit history for leaked credentials.
- **Multi-Modal Forensics:** Real-world incident investigations require correlating evidence across disparate mediums: email metadata, network packets, visual files, audio spectrograms, and version control objects.
