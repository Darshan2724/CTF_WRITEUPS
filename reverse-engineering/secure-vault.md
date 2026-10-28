# KPMG CTF 2026: Secure Vault — Reverse Engineering Writeup

## Challenge Overview

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Secure Vault |
| **Category** | Reverse Engineering / Binary Analysis |
| **Points** | 20 |
| **Target Binary** | `challenge1` (64-bit x86-64 ELF executable) |
| **Flag Format** | `KPMG_CTF{...}` |
| **Flag** | `KPMG_CTF{5EjHKiHi8fOv1Nq6ea0ite9wzMxeZuMSOPNAI9ARuYQ}` |

---

## 1. Challenge Description

The Secure Data Vault appears to be protected by a SHA-256 password prompt. However, the challenge description hints that this visible prompt is a decoy and that the actual secret is concealed within the binary's internal ELF structure.

The objective is to reverse-engineer the executable, investigate why normal section analysis fails, uncover the hidden metadata routine, and reconstruct the flag.

---

## 2. Tools Used

- `file` — Inspect binary architecture and format
- `readelf` — Parse ELF file headers and program header tables
- `strings` — Extract static string literals
- `llvm-objdump` / `objdump` — Disassemble machine code instructions
- **Python 3** — Reconstruct and execute the decoding algorithm
- **Linux Terminal** — Binary execution and argument testing

---

## 3. Initial Binary Analysis

Inspecting the file format:

```bash
file challenge1
```

**Output:**
```text
challenge1: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, no section header table
```

The output confirms a 64-bit little-endian ELF Position-Independent Executable (PIE), with a crucial detail: **no section header table**. 

Standard tools like `objdump -s -j .rodata` fail because section names do not exist. However, the ELF loader relies on **program headers (segments)**, not section headers, to execute the program.

Inspecting the ELF header with `readelf -h challenge1`:
- **Class:** ELF64
- **Endianness:** Little-endian
- **Type:** DYN (PIE executable)
- **Entry point:** `0x14a0`
- **Program header offset (`e_phoff`):** `0x40` (Entry size: 56 bytes, Count: 13)
- **Section header offset (`e_shoff`):** `0x0` (Count: 0)

---

## 4. Identifying the Distraction

Running `strings -a challenge1` reveals:
```text
SECURE DATA VAULT
STATUS: LOCKED
ENCRYPTION: SHA-256
ENTER PASSWORD >
ERROR: INVALID PASSWORD.
Calculated Hash: %s
Intrusion logged.
correct
wrong
challenge metadata is damaged
```

The binary contains a standard password prompt that computes a SHA-256 digest. However, the strings also contain the error `challenge metadata is damaged`. This hints that passing arguments on the command line triggers an alternate verification path.

---

## 5. Analyzing the Program Headers

Inspecting the program headers with `readelf -lW challenge1`:

Each ELF64 program header entry is 56 bytes. The executable contains 13 program headers (indices 0 through 12). Examining the physical address (`p_paddr`, offset `0x18` in the 56-byte header) reveals abnormal 64-bit values in the last 7 entries:

| Program Header Type | Physical Address (`p_paddr`) Value |
| :--- | :--- |
| `DYNAMIC` | `0xe4eab6df725f4d76` |
| `NOTE` | `0x004ecf7dd3a8996b` |
| `NOTE` | `0xe8a2432056262590` |
| `GNU_PROPERTY` | `0x6b029a8712068126` |
| `GNU_EH_FRAME` | `0xd4832d32ade1e6cb` |
| `GNU_STACK` | `0x81e587a7666c70c9` |
| `GNU_RELRO` | `0x0000df6af6b9ff69` |

In typical user-space Linux executables, `p_paddr` is unused (set to 0 or equal to `p_vaddr`). Here, the last 7 headers store 56 bytes (7 × 8 bytes) of encoded payload!

---

## 6. Disassembling the Decoding Routine

Disassembling the validation function starting at `0x1220`:

```bash
llvm-objdump -d --x86-asm-syntax=intel --start-address=0x1220 --stop-address=0x1495 challenge1
```

The routine executes the following steps:
1. **ELF Verification:** Validates the ELF magic bytes, 64-bit class, 56-byte program header size, and verifies that `e_phnum >= 7`.
2. **Buffer Extraction:** Concatenates the 64-bit `p_paddr` values of the last 7 program headers in little-endian order to produce a 56-byte buffer.
3. **Key Derivation:**
   - Initializes `key = (e_entry_low ^ e_phnum ^ 0x37) & 0xFF`.
   - Iterates through all program headers. For each `PT_LOAD` (type 1) segment:
     - `key ^= ((p_flags & 0xFF) * counter) & 0xFF` (where `counter` starts at 3 and increments).
     - `key ^= (p_align >> 12) & 0xFF`.
   - Finally XORs `key` with `0xA5`.
4. **Length Calculation:** Derives the payload length: `length = (key ^ uint32(encoded[0:4]) ^ 0x5A) & 0xFF` (evaluates to 53 bytes).
5. **Byte Transformation:**
   For index `i` from 0 to 52:
   - Takes byte `encoded[i + 1]`.
   - XORs with `(0x29 + 0x13 * i) & 0xFF`.
   - XORs with `ROL8(key, i & 7)` (left-rotated key).
6. **Reversal:** Reverses the decoded byte array to produce the final ASCII string.

---

## 7. Python Solver Implementation

```python
import struct
import sys

def rol8(value, shift):
    """Rotate an 8-bit integer to the left."""
    shift &= 7
    if shift == 0:
        return value & 0xFF
    return ((value << shift) | (value >> (8 - shift))) & 0xFF

path = sys.argv[1] if len(sys.argv) > 1 else "challenge1"
with open(path, "rb") as f:
    data = f.read()

# Verify ELF64
if data[:4] != b"\x7fELF" or data[4] != 2 or data[5] != 1:
    raise SystemExit("Expected a little-endian ELF64 binary")

# Read ELF header
e_phoff = struct.unpack_from("<Q", data, 0x20)[0]
e_phentsize = struct.unpack_from("<H", data, 0x36)[0]
e_phnum = struct.unpack_from("<H", data, 0x38)[0]

# Parse all program headers
phdrs = [
    struct.unpack_from("<IIQQQQQQ", data, e_phoff + i * e_phentsize)
    for i in range(e_phnum)
]

# Build 56-byte encoded buffer from last 7 p_paddr fields (index 4 in unpack)
encoded = b"".join(struct.pack("<Q", ph[4]) for ph in phdrs[-7:])

# Derive the 8-bit key
key = (data[0x18] ^ e_phnum ^ 0x37) & 0xFF
counter = 3
for ph in phdrs:
    p_type, p_flags, _, _, _, _, _, p_align = ph
    if p_type == 1:  # PT_LOAD
        key ^= ((p_flags & 0xFF) * counter) & 0xFF
        key ^= (p_align >> 12) & 0xFF
        key &= 0xFF
        counter += 1
key = (key ^ 0xA5) & 0xFF

# Calculate payload length
length = (key ^ struct.unpack_from("<I", encoded, 0)[0] ^ 0x5A) & 0xFF

# Decode bytes
stage = bytearray()
for i in range(length):
    val = (
        encoded[i + 1]
        ^ ((0x29 + 0x13 * i) & 0xFF)
        ^ rol8(key, i & 7)
    )
    stage.append(val)

# Reverse decoded bytes
flag = bytes(reversed(stage)).decode("ascii")

print(f"Program headers: {e_phnum}")
print(f"Derived key: 0x{key:02x}")
print(f"Decoded length: {length} bytes")
print(f"Flag: {flag}")
```

---

## 8. Solver Execution & Local Verification

Running the solver:

```bash
python3 solve.py challenge1
```

**Output:**
```text
Program headers: 13
Derived key: 0x19
Decoded length: 53 bytes
Flag: KPMG_CTF{5EjHKiHi8fOv1Nq6ea0ite9wzMxeZuMSOPNAI9ARuYQ}
```

Verifying against the binary:

```bash
./challenge1 'KPMG_CTF{5EjHKiHi8fOv1Nq6ea0ite9wzMxeZuMSOPNAI9ARuYQ}'
# Output: correct
```

---

## 9. Final Flag

```text
KPMG_CTF{5EjHKiHi8fOv1Nq6ea0ite9wzMxeZuMSOPNAI9ARuYQ}
```

---

## 10. Key Takeaways
- **Program Headers vs Section Headers:** Striping section headers stops tools like `readelf -S` or basic debuggers, but the OS only requires program headers (`PT_LOAD`) to execute code.
- **Header Field Concealment:** Unused header fields like `p_paddr` in ELF binaries can store arbitrary payloads.
- **Deceiving User Interfaces:** Prompts, hashes, and decoy password checks are common CTF distractions. Inspecting argument handlers and code paths often exposes the actual verification routine.
