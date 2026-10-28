# KPMG CTF 2026: Corrupted Vision — Steganography & Forensics Writeup

## Challenge Overview

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Corrupted Vision |
| **Category** | Steganography / File Forensics |
| **Points** | 30 |
| **Target Artifact** | `challenge_file` |
| **Flag** | `KPMG_CTF{S33K_AND_F1ND_M3}` |

---

## 1. Challenge Description

> *"We intercepted a file our tools can't classify. It looks like a simple profile picture, but we know there's more to it. Expand your view to uncover the hidden payload."*

The objective was to investigate the provided artifact, identify and repair corrupted file headers, resolve dimensional truncation preventing normal rendering, and decode the hidden payload embedded in the revealed image content.

---

## 2. Tools Used

- **Kali Linux**
- `file` — Identify file signatures and mime types
- `xxd` / `od` — Hexadecimal stream and header inspection
- **Python 3** — Scripting header restoration and height expansion
- **Image Viewer** (`feh` / `eog`) — Visual inspection of the recovered image

---

## 3. Step-by-Step Solution

### Step 1: Initial File & Signature Analysis
Running the standard `file` utility returned a generic data classification:

```bash
file challenge_file
# Output: challenge_file: data
```

Inspected the first 32 bytes using `xxd`:

```bash
xxd -l 32 challenge_file
```

The file began with the signature:
```text
89 50 4e 47 0d 0a 1a 0a
```

While the magic bytes corresponded to a PNG image header (`.PNG\r\n\x1a\n`), examining the data immediately downstream revealed standard JPEG/JFIF markers (such as quantization tables `FF DB` and start of frame markers `FF C0`) rather than PNG chunks (`IHDR`, `IDAT`, `IEND`).

**Finding:** The file was a valid JPEG image whose original file signature had been deliberately replaced with a fake PNG header, preventing image decoders from loading it.

---

### Step 2: Repairing the JPEG Header & Image Dimensions
To restore the JPEG header, the first 8 bytes needed to be replaced with the standard JPEG/JFIF magic bytes:
```text
FF D8 FF E0 00 10 4A 46
```

Furthermore, the challenge prompt advised to *"Expand your view to uncover the hidden payload"*. In JPEG files, the image height and width are defined inside the **SOF0 (Start of Frame)** segment, marked by `FF C0`.
- The SOF0 segment format is:
  - `FF C0` (Marker)
  - `00 11` (Segment length)
  - `08` (Precision bits per sample)
  - Next 2 bytes: **Image Height** (Big-Endian)
  - Next 2 bytes: **Image Width** (Big-Endian)

Inspecting the SOF0 marker showed that the stored height was only **550 pixels**, despite significantly more compressed scan data existing in the file. Increasing the stored height to **1024 pixels** would force the decoder to render the hidden lower section.

I wrote the following Python repair script:

```python
from pathlib import Path

# Load raw corrupted file
data = bytearray(Path("challenge_file").read_bytes())

# Restore JPEG/JFIF magic header (8 bytes)
data[:8] = bytes.fromhex("FFD8FFE000104A46")

# Locate Start of Frame (SOF0) marker
offset = data.find(bytes.fromhex("FFC0"))
if offset == -1:
    raise ValueError("JPEG SOF0 marker not found")

# Modify stored image height at offset+5 to 1024 pixels
data[offset + 5 : offset + 7] = (1024).to_bytes(2, "big")

# Save repaired image
Path("repaired.jpg").write_bytes(data)
print("Repaired image saved as repaired.jpg")
```

Verified the repaired file:

```bash
file repaired.jpg
# Output: repaired.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, baseline, precision 8, 800x1024, components 3
```

---

### Step 3: Discovering the Hidden Message
Opening `repaired.jpg` revealed the original profile picture on top, followed by an additional paragraph of text across the bottom section that had previously been cropped out by the altered height value.

The text was composed of deliberate word structures. Analyzing the length of each word revealed a clear encoding rule:

| Word Length | Morse Code Meaning |
| :--- | :--- |
| **3 letters** | Character separator (space between letters) |
| **5 letters** | Dot (`.`) |
| **7 letters** | Dash (`-`) |

---

### Step 4: Morse Code Decoding

Applying the word-length mapping to the paragraph produced the following sequence:

| Decoded Morse Pattern | Decoded Character |
| :--- | :--- |
| `-.-` | **K** |
| `.--.` | **P** |
| `--` | **M** |
| `--.` | **G** |
| `..--.-` | **_** |
| `-.-.` | **C** |
| `-` | **T** |
| `..-.` | **F** |
| `-.--.` | **(** |
| `...` | **S** |
| `...--` | **3** |
| `...--` | **3** |
| `-.-` | **K** |
| `..--.-` | **_** |
| `.-` | **A** |
| `-.` | **N** |
| `-..` | **D** |
| `..--.-` | **_** |
| `..-.` | **F** |
| `.----` | **1** |
| `-.` | **N** |
| `-..` | **D** |
| `..--.-` | **_** |
| `--` | **M** |
| `...--` | **3** |
| `-.--.-` | **)** |

The raw decoded string was:
```text
KPMG_CTF(S33K_AND_F1ND_M3)
```

Converting the outer parentheses into conventional curly braces (`{}`) yielded the valid flag.

---

## 4. Final Flag

```text
KPMG_CTF{S33K_AND_F1ND_M3}
```

---

## 5. Key Takeaways
1. **Never trust file extensions or top-level file signatures blindly:** Attackers and challenge authors often disguise file types by substituting header bytes.
2. **JPEG Structure Manipulation:** The SOF0 marker dictates image dimensions. Modifying height bytes allows steganographic data hiding in plain sight.
3. **Word-Length Steganography:** When text in an image appears slightly unnatural or grammatically repetitive, check for patterns in word lengths or character counts.
