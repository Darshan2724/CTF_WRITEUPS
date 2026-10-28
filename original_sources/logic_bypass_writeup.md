# CTF Write-up: Logic Bypass

## Challenge overview

- **Challenge:** Logic Bypass
- **Category:** Reverse Engineering / Binary Analysis
- **Artifact:** `challenge` (64-bit Linux ELF executable)
- **Flag format:** `KPMG_CTF{...}`

The provided `challenge` file is a Linux executable. Its strings describe a local authentication check and say the goal is to make the binary approve access. The program normally prints the flag only after a submitted key passes several checks. Rather than brute-forcing the access key, we can statically inspect the executable and reproduce the small decoding routine used to recover the flag.

## 1. Identify the file

Start by checking the file type:

```bash
file challenge
```

The output identifies it as a 64-bit x86-64 ELF executable, dynamically linked and **not stripped**. Because it is not stripped, many internal symbol names remain available, which makes the binary easier to inspect.

Calculate a hash if you want to document the exact artifact analyzed:

```bash
sha256sum challenge
```

For this copy, the SHA-256 is:

```text
fa45ae5f2a7e80a26d5f2a9ce690f969856497d9edcd0497fa2497f50c8f0b60
```

## 2. Look for strings and useful symbols

Inspect printable strings:

```bash
strings -a -t x challenge
```

Notable strings include:

- `LOGIC BYPASS // AUTH`
- `A secret is protected by a local check.`
- `Your goal is to make the binary approve access.`
- `Enter access key:`
- `[ACCESS DENIED]`
- `[ACCESS GRANTED]`
- `Flag: %s`

Next, inspect the symbol table:

```bash
nm -C challenge | grep -E 'FLAG_KEY|ENC_FLAG|decode_flag|validate_password|transform_password|looks_reasonable|secondary_decoy|weird_mix|fnv1a32'
```

The important symbols include:

| Symbol | Role |
|---|---|
| `FLAG_KEY` | Repeating XOR key stored in the executable |
| `ENC_FLAG` | Encoded flag bytes |
| `decode_flag` | Routine that decodes those bytes |
| `validate_password` | Main access-key validation routine |
| `looks_reasonable` | Checks basic input requirements |
| `transform_password` | Produces a 16-byte transformed value |
| `fnv1a32` / `weird_mix` | Hashing and additional validation calculations |
| `secondary_decoy` | Additional logic that complicates analysis but is not needed to decode the flag |

These names suggest two possible avenues: reverse the authentication calculation or examine how the flag itself is stored. The latter is considerably simpler.

## 3. Understand the program flow

Disassemble `main`:

```bash
objdump -d -M intel --disassemble=main challenge
```

The relevant flow is:

1. Print the banner and prompt for an access key.
2. Read input with `fgets` and remove the newline using `strcspn`.
3. Pass the input to `validate_password`.
4. If validation fails, print `[ACCESS DENIED]` and the hint that the decision is made by logic rather than a memory-corruption bug.
5. If validation succeeds, print `[ACCESS GRANTED]`, call `decode_flag`, and print the decoded string.

The input checker requires a length in the range 6–48, at least one alphabetic character, at least one digit, and no whitespace. The more complicated validation path transforms the input, compares the result against a fixed 16-byte value, and checks a hash-derived value against a constant. It also runs extra mixing logic and mutates a global variable named `analysis_noise`.

Those checks explain why a random input is rejected. However, they only gate the program's *printing* of the flag. They do not make the encoded flag or its decoding key secret from someone who can examine the executable.

## 4. Find the encoded flag and XOR key

Dump the read-only data section:

```bash
objdump -s -j .rodata challenge
```

The symbol table places `FLAG_KEY` at virtual address `0x402020` and `ENC_FLAG` at `0x402040`.

The eight-byte XOR key is:

```text
5a 17 c0 de 13 37 42 69
```

The 53 encoded flag bytes are:

```text
11478d994c74162f212595bf3e7f1a441d4db291604d380c2f66f8b971653a31194281b15e6628110c43a5ec761a1d391e2386876e
```

The data is not encrypted with a strong cryptographic scheme. The key and encoded text are both embedded in the same executable, so the operation can be reproduced offline.

## 5. Reverse the `decode_flag` routine

Disassemble the decoder:

```bash
objdump -d -M intel --disassemble=decode_flag challenge
```

The important instructions show that each byte from `ENC_FLAG` is XORed with a byte from `FLAG_KEY`. The key index is calculated with `i & 7`, meaning the eight-byte key repeats every eight bytes. In pseudocode, the routine does:

```text
for i from 0 to 52:
    decoded[i] = ENC_FLAG[i] XOR FLAG_KEY[i mod 8]
decoded[53] = '\0'
```

The equivalent equation is:

\[
P_i = C_i \oplus K_{i \bmod 8}
\]

where `C` is the encoded byte array, `K` is the eight-byte key, and `P` is the decoded output. XOR is its own inverse, so applying the same key to each encoded byte recovers the original character.

There is a small size check in `decode_flag`: it decodes the fixed 53-byte value when called with a buffer size greater than 53. `main` calls it with size `0x36` (54), which satisfies that check. For static recovery, the same operation can simply be applied to the bytes directly.

## 6. Decode the flag with Python

Run the following script. It does not execute the unknown binary; it only applies the XOR operation to the constants recovered from `.rodata`.

```python
key = bytes.fromhex("5a17c0de13374269")
encoded = bytes.fromhex(
    "11478d994c74162f212595bf3e7f1a44"
    "1d4db291604d380c2f66f8b971653a31"
    "194281b15e6628110c43a5ec761a1d39"
    "1e2386876e"
)

flag = bytes(byte ^ key[i % len(key)] for i, byte in enumerate(encoded))
print(flag.decode("ascii"))
```

Output:

```text
KPMG_CTF{2Ua-HX-GZrOszzeuq8gbRxXCUAoMQjxVTe2e-_PD4FY}
```

## 7. Why the validation logic can be skipped for flag recovery

The authentication path is deliberately noisy. `looks_reasonable` checks basic input properties, `transform_password` mixes the input bytes with a value derived from FNV-1a and a constant table, and `validate_password` compares that 16-byte result against a stored expected value. It also combines the FNV-1a result with `weird_mix` and checks a final constant.

The `secondary_decoy` function performs more calculations and may rotate `analysis_noise`. That global mutation is not part of the final acceptance comparison, so it is a distraction rather than a necessary step in decoding the flag.

For a challenge focused specifically on producing an input that reaches `[ACCESS GRANTED]`, these conditions would need to be satisfied or bypassed at runtime. For the objective of recovering the flag from the supplied artifact, however, static analysis is enough: `ENC_FLAG`, `FLAG_KEY`, and the XOR algorithm are all present in the binary. No brute force, memory corruption, or password guessing is required.

## 8. Final flag

```text
KPMG_CTF{2Ua-HX-GZrOszzeuq8gbRxXCUAoMQjxVTe2e-_PD4FY}
```

## Key takeaways

- Check the file type before choosing a reversing approach.
- `strings`, `nm`, and `objdump` are useful first-line tools for an unstripped ELF.
- Do not focus exclusively on a complicated validation routine; inspect how the protected data is stored and decoded.
- A reversible XOR transformation offers no meaningful secrecy when both the key and ciphertext are embedded in the same executable.
