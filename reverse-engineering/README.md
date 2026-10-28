# Reverse Engineering & Binary Analysis Writeups

This directory contains writeups for compiled binary analysis and reverse engineering challenges from KPMG CTF 2026 Campus Edition.

## Challenge Index

| Challenge | Points | Architecture / Type | Key Techniques | Final Flag |
| :--- | :--- | :--- | :--- | :--- |
| [Secure Vault](secure-vault.md) | 20 | 64-bit ELF (PIE, no section headers) | Program header table parsing, physical address (`p_paddr`) extraction, custom rotating key derivation | `KPMG_CTF{5EjHKiHi8fOv1Nq6ea0ite9wzMxeZuMSOPNAI9ARuYQ}` |
| [Algebraic Nightmare](algebraic-nightmare.md) | 30 | 64-bit ELF (stripped, custom VM) | Custom virtual machine (ANMBA2), bytecode XOR decoding, backward dependency slicing (14,000 → 662 ops), modular arithmetic inversion | `KPMG_CTF{xUXOkGYKxH_0HVyXmwEKhUIodKep12X0dtHmzzhIEmk}` |
| [Logic Bypass](logic-bypass.md) | 20 | 64-bit ELF (unstripped) | Symbol table inspection, `.rodata` key/ciphertext extraction, authentication logic bypass | `KPMG_CTF{2Ua-HX-GZrOszzeuq8gbRxXCUAoMQjxVTe2e-_PD4FY}` |
