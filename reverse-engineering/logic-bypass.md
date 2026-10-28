# KPMG CTF 2026: Logic Bypass — Reverse Engineering Writeup

## Challenge Overview

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Logic Bypass |
| **Category** | Reverse Engineering / Binary Analysis |
| **Points** | 20 |
| **Artifact** | `challenge` (64-bit Linux x86-64 unstripped ELF executable) |
| **Flag Format** | `KPMG_CTF{...}` |
| **Flag** | `KPMG_CTF{2Ua-HX-GZrOszzeuq8gbRxXCUAoMQjxVTe2e-_PD4FY}` |

---

## 1. Challenge Description

The provided `challenge` file is an x86-64 Linux executable. Its printable strings describe a local authentication check requiring a valid access key. The program normally prints the flag only after a submitted key passes several complex validation routines. Rather than reversing or brute-forcing the access key, static inspection reveals that the encoded flag and its decoding key are stored in plaintext within the binary.

---

## 2. Tools Used

- `file` — Identify binary architecture and linking properties
- `sha256sum` — Cryptographic hash verification of the binary
- `strings` — Extract printable string literals and addresses
- `nm` — Extract symbols from the unstripped symbol table
- `objdump` — Disassemble functions and inspect `.rodata`
- **Python 3** — Offline static decryption

---

## 3. Step-by-Step Solution

### Step 1: File Identification & Triage

Checking the binary type:

```bash
file challenge
# Output: challenge: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, not stripped
```

Because the binary is **not stripped**, internal symbol names and function identifiers are preserved.

SHA-256 Hash of the artifact:
```text
fa45ae5f2a7e80a26d5f2a9ce690f969856497d9edcd0497fa2497f50c8f0b60
```

---

### Step 2: Symbol Analysis

Inspecting the symbol table using `nm`:

```bash
nm -C challenge | grep -E 'FLAG_KEY|ENC_FLAG|decode_flag|validate_password|transform_password|looks_reasonable|secondary_decoy|weird_mix|fnv1a32'
```

Key symbols uncovered:

| Symbol | Role in Binary |
| :--- | :--- |
| `FLAG_KEY` | Repeating 8-byte XOR key located in `.rodata` |
| `ENC_FLAG` | 53-byte encoded flag payload |
| `decode_flag` | Routine that applies the XOR transformation to print the flag |
| `validate_password` | Main authentication routine |
| `transform_password` | Produces a 16-byte intermediate hash |
| `fnv1a32` / `weird_mix` | Decoy hashing logic |
| `secondary_decoy` | Obfuscation logic that mutates `analysis_noise` |

---

### Step 3: Flow Analysis (`main`)

Disassembling `main`:

```bash
objdump -d -M intel --disassemble=main challenge
```

1. Prints the banner `LOGIC BYPASS // AUTH` and prompts for an access key.
2. Reads user input via `fgets` and strips the newline.
3. Passes the input to `validate_password`.
4. If invalid: prints `[ACCESS DENIED]` and exits.
5. If valid: prints `[ACCESS GRANTED]`, invokes `decode_flag`, and prints `Flag: %s`.

Crucially, the password check only gates the *execution* of `decode_flag`. It does not encrypt or protect the actual ciphertext or key.

---

### Step 4: Extracting `.rodata` Artifacts

Dumping `.rodata` section:

```bash
objdump -s -j .rodata challenge
```

- `FLAG_KEY` (virtual address `0x402020`):
  ```text
  5a 17 c0 de 13 37 42 69
  ```
- `ENC_FLAG` (virtual address `0x402040`):
  ```text
  11478d994c74162f212595bf3e7f1a441d4db291604d380c2f66f8b971653a31194281b15e6628110c43a5ec761a1d391e2386876e
  ```

---

### Step 5: Reverse Engineering `decode_flag`

Disassembling `decode_flag`:

```bash
objdump -d -M intel --disassemble=decode_flag challenge
```

The loop runs for 53 bytes, computing:
$$P_i = C_i \oplus K_{i \bmod 8}$$

Because XOR is self-inverting, applying `FLAG_KEY` directly to `ENC_FLAG` reconstructs the plaintext flag immediately.

---

### Step 6: Python Offline Solver

```python
key = bytes.fromhex("5a17c0de13374269")
encoded = bytes.fromhex(
    "11478d994c74162f212595bf3e7f1a44"
    "1d4db291604d380c2f66f8b971653a31"
    "194281b15e6628110c43a5ec761a1d39"
    "1e2386876e"
)

flag = bytes(byte ^ key[i % len(key)] for i, byte in enumerate(encoded))
print("Recovered Flag:", flag.decode("ascii"))
```

**Solver Output:**
```text
Recovered Flag: KPMG_CTF{2Ua-HX-GZrOszzeuq8gbRxXCUAoMQjxVTe2e-_PD4FY}
```

---

## 4. Final Flag

```text
KPMG_CTF{2Ua-HX-GZrOszzeuq8gbRxXCUAoMQjxVTe2e-_PD4FY}
```

---

## 5. Key Takeaways
- **Unstripped Binaries:** Symbol names in unstripped binaries often immediately reveal the high-value functions and global variables.
- **Bypassing Noisy Validation:** In many binary challenges, reverse engineers spend time analyzing complex hashing logic (`fnv1a32`, custom mixing, password transformation) that is entirely bypassed if the flag decoding routine is self-contained.
- **Embedded Keys:** Symmetrical XOR operations embedded alongside ciphertext in static data provide zero confidentiality against static analysis.
