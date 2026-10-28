# KPMG CTF 2026: Algebraic Nightmare — Custom VM Bytecode Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Algebraic Nightmare |
| **Category** | Reverse Engineering / Virtual Machine Obfuscation |
| **Difficulty** | Advanced |
| **Points** | 30 |
| **Artifact** | `challenge.elf` (64-bit x86-64 ELF stripped executable) |
| **Flag Format** | `KPMG_CTF{...}` |
| **Flag** | `KPMG_CTF{xUXOkGYKxH_0HVyXmwEKhUIodKep12X0dtHmzzhIEmk}` |

---

## 1. Challenge Overview

The challenge presents a compiled ELF binary named `challenge.elf`. The scenario states that an advanced malware sample uses thousands of complex arithmetic and Boolean operations instead of a standard encryption routine.

The objective is to perform static and dynamic analysis on the binary, identify the execution model, unravel the mathematical checks, and recover the 56-byte accepted specimen.

---

## 2. Initial Binary Triage

Inspecting the file properties:

```bash
file challenge.elf
# Output: challenge.elf: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, stripped
```

Running `strings -a challenge.elf | grep -E "specimen|accepted|rejected"` yields:
```text
specimen> 
accepted
rejected
```

Examining the disassembly around `main` reveals:
1. Prompts for input using `specimen> `.
2. Reads candidate string via `fgets` and strips newlines using `strcspn`.
3. Verifies that the input string length matches the required size (56 bytes).
4. Invokes a dispatcher routine that interprets an encoded memory blob.

---

## 3. Bytecode Structure & Header Decoding

The binary contains an obfuscated data blob embedded at file offset `0x2040` spanning `0x4462c` bytes. A repeating 8-byte XOR key is stored at offset `0x46670`:
```text
78 d0 13 6c 8e d2 71 fb
```

I wrote a Python script to decode the bytecode blob:

```python
import struct

with open("challenge.elf", "rb") as f:
    binary = f.read()

blob_offset = 0x2040
key_offset = 0x46670
blob_size = 0x4462c

key = binary[key_offset : key_offset + 8]
encoded = binary[blob_offset : blob_offset + blob_size]
decoded = bytes(byte ^ key[i & 7] for i, byte in enumerate(encoded))

magic = decoded[:8]
num_instructions, num_checks, input_len = struct.unpack_from("<III", decoded, 8)

print("Magic:", magic)
print("Instructions:", num_instructions)
print("Validation checks:", num_checks)
print("Required input length:", input_len)
```

**Output:**
```text
Magic: b'ANMBA2\x00\x00'
Instructions: 14000
Validation checks: 7
Required input length: 56
```

The header confirms a custom Virtual Machine titled **ANMBA2**, containing **14,000 instructions**, checking **7 output conditions**, and processing an **exact 56-byte input**.

---

## 4. Virtual Machine Architecture & Instruction Set

Each VM instruction is 20 bytes long, comprising an opcode, source/destination registers, and immediate operands. By reverse engineering the C dispatcher loop in `challenge.elf`, the opcode operations were cataloged:

| Opcode | Operation | Semantic Description |
| :---: | :--- | :--- |
| `0` | Const | Load mixed 64-bit immediate constant |
| `1` | Zero | Return zero / clear register |
| `2` | Load Input | Load one of 7 64-bit words from the 56-byte input (`input[word_idx]`) |
| `3` | Add | 64-bit addition (modulo $2^{64}$) |
| `4` | Sub | 64-bit subtraction (modulo $2^{64}$) |
| `5` | XOR | Bitwise XOR |
| `6` | AND | Bitwise AND |
| `7` | OR | Bitwise OR |
| `8` | NOT | Bitwise NOT |
| `9` | SHL | Logical left shift |
| `10` | SHR | Logical right shift |
| `11` | ROL | 64-bit rotate left |
| `12` | ROR | 64-bit rotate right |
| `13` | Mul | 64-bit multiplication (modulo $2^{64}$) |
| `14` | Neg | Arithmetic negation (two's complement) |

---

## 5. Backward Dependency Reduction (14,000 → 662 Instructions)

The decoded footer specifies 7 output registers that must match 7 target 64-bit constants:
```text
Output Register Indices: [486, 518, 550, 582, 614, 646, 678]
```

Rather than attempting to forward-solve all 14,000 instructions (which contains dead code and decoys), I performed a **backward dependency analysis** starting from these 7 target registers.

- Tracing register definitions recursively revealed that only **~662 instructions** actually influence the 7 validation outputs!
- Over 95% of the instruction stream consisted of unreachable or discarded arithmetic operations designed to overwhelm static decompilation.

---

## 6. Mathematical Inversion & Flag Reconstruction

Each of the 7 output checks corresponds to a chain of reversible operations applied to one of the seven 64-bit words of the input:
- **XOR:** Self-inverting ($A \oplus B \oplus B = A$).
- **Addition/Subtraction:** Inverted via standard modulo $2^{64}$ arithmetic ($A = (C - B) \pmod{2^{64}}$).
- **Rotations:** Inverted by rotating in the opposite direction by the same bit count.
- **Multiplications:** Since multiplications in the reduced graph used odd multipliers, they are invertible modulo $2^{64}$ using the modular multiplicative inverse:
  $$\text{inv}(k) \times k \equiv 1 \pmod{2^{64}}$$

Using the known prefix `KPMG_CTF{` to seed and verify the algebraic constraints, I reconstructed the 7 64-bit integers and converted them into ASCII:

- Character count of recovered flag: **53 characters**.
- Since the binary strictly requires a 56-byte input, the remaining 3 bytes are trailing underscore padding (`___`).

---

## 7. Verification Against the Binary

Testing the 56-byte padded string directly against `challenge.elf`:

```bash
printf '%s\n' 'KPMG_CTF{xUXOkGYKxH_0HVyXmwEKhUIodKep12X0dtHmzzhIEmk}___' | ./challenge.elf
```

**Output:**
```text
specimen> accepted
```

Testing the unpadded flag:

```bash
printf '%s\n' 'KPMG_CTF{xUXOkGYKxH_0HVyXmwEKhUIodKep12X0dtHmzzhIEmk}' | ./challenge.elf
```

**Output:**
```text
specimen> rejected
```

This confirms that the 3 trailing underscores are padding required by the binary's 56-byte `fgets` length gate, while the core payload inside the curly braces is the official CTF flag.

---

## 8. Final Flag

```text
KPMG_CTF{xUXOkGYKxH_0HVyXmwEKhUIodKep12X0dtHmzzhIEmk}
```

---

## 9. Key Takeaways
1. **VM Bytecode Identification:** Look for repetitive interpreter dispatch loops reading structured structs from encoded data sections.
2. **Backward Slicing / Dependency Pruning:** Never get bogged down analyzing the entirety of a large obfuscated instruction set; start from the final comparison and slice backwards to isolate relevant operations.
3. **Reversible Modular Arithmetic:** Most arithmetic operators (addition, subtraction, XOR, rotations, and odd-integer multiplications) are completely deterministic and bi-directional over 64-bit word rings.
