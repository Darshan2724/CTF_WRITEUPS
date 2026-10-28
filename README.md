# 🛡️ KPMG CTF 2026 Campus Edition — Writeups & Solutions

[![CTF](https://img.shields.io/badge/CTF-KPMG%20Campus%20Edition%202026-blue?style=for-the-badge&logo=target)](https://kicyber.ctfd.io/)
[![Platform](https://img.shields.io/badge/Platform-KICyber%20CTFd-orange?style=for-the-badge&logo=securityscorecard)](https://kicyber.ctfd.io/)
[![Categories](https://img.shields.io/badge/Categories-Web%20%7C%20Rev%20%7C%20Pwn%20%7C%20Forensics%20%7C%20Crypto%20%7C%20AI%20%7C%20OT-green?style=for-the-badge)](.)
[![Solves](https://img.shields.io/badge/Status-100%25%20Documented-brightgreen?style=for-the-badge)](.)

Welcome to the comprehensive, structured repository of technical writeups, exploit scripts, and methodology notes for **KPMG CTF 2026 Campus Edition** (hosted on the KICyber CTFd platform).

Each writeup contains a thorough breakdown of the initial problem, vulnerability analysis, step-by-step reproduction ("How I Solved It"), exploit scripts/payloads, captured flags, and defensive hardening takeaways.

---

## 📑 Challenge Index Matrix

| # | Challenge Name | Category | Points / Diff | Key Vulnerability / Technique | Writeup Link | Flag |
| :-: | :--- | :--- | :-: | :--- | :-: | :--- |
| **01** | **First Flag** | Misc / Recon | 10 pts | Documentation footer inspection | [View Writeup](misc/welcome-and-guidelines.md#challenge-1-first-flag-ctf-guidelines) | `KPMG_CTF{g00d_luck_w1tH_th3_CTF!!}` |
| **02** | **Welcome Flag** | Web / Recon | 10 pts | Web crawler directive analysis (`robots.txt`) | [View Writeup](misc/welcome-and-guidelines.md#challenge-2-welcome-flag-robotstxt) | `KPMG_CTF{w3lc0me_t0_kicyberctf_2026!!}` |
| **03** | **Onion** | Cryptography | 10 pts | Multi-stage pipeline: Binary → Base64 → Hex → ASCII → Atbash | [View Writeup](cryptography/onion.md) | `KPMG_CTF{p33l_th3_l4y3rs_l1k3_4n_0n10n}` |
| **04** | **Web Troll Clock** | Web Exploitation | 20 pts | Cosmetic client-side OTP bypass & WebSocket manipulation | [View Writeup](web/web-troll-clock.md) | `KPMG_CTF{0U1gjUKP8iZUwkPUBq01yli5XZr7IFKUV2grCEWx4HY}` |
| **05** | **VaultDocs - 1** | Web Exploitation | 10 pts | LFI path traversal (`/etc/passwd`), default SSH credentials | [View Writeup](web/vaultdocs-1.md) | `KPMG_CTF{2R68vP6s4_AUfTIldzjytnHuN76JGJn8MXL4IKy0R6k}` |
| **06** | **VaultDocs - 2** | Privilege Escalation | 10 pts | Sudoers `/bin/tar` GTFOBins checkpoint command execution to Root | [View Writeup](privilege-escalation/vaultdocs-2.md) | `KPMG_CTF{vujzSbEPjuGNy7fw7JMzFwgXuk21JyU8zB2fIaDqGmg}` |
| **07** | **AI SecureBot Support** | AI Security | 20 pts | QA Evaluator roleplay jailbreak & JSON schema enforcement | [View Writeup](ai-security/securebot-and-excessive-agency.md#challenge-1-ai-securebot-support) | `KPMG_CTF{Ulcpla31x2GGPqhybX6d0wpjhecBs5GVIJKhcJzJ5EE}` |
| **08** | **AcmeBank Assistant** | AI Security | 30 pts | LLM Excessive Agency / Parameterized tool invocation IDOR | [View Writeup](ai-security/securebot-and-excessive-agency.md#challenge-2-acmebank-assistant-excessive-agency) | `KPMG_CTF{XbGEyIlxVPj65COEujnrzMAx4Yf5bUrjPeg6ywd11cE}` |
| **09** | **Corrupted Vision** | Steganography / Forensics | 30 pts | File header repair (PNG to JPEG), SOF0 height fix, Morse code | [View Writeup](forensics-stego/corrupted-vision.md) | `KPMG_CTF{S33K_AND_F1ND_M3}` |
| **10** | **Secure Vault** | Reverse Engineering | 20 pts | Stripped section table ELF analysis, `p_paddr` buffer solver | [View Writeup](reverse-engineering/secure-vault.md) | `KPMG_CTF{5EjHKiHi8fOv1Nq6ea0ite9wzMxeZuMSOPNAI9ARuYQ}` |
| **11** | **Westbound** | Forensics / OSINT | 30 pts | Phishing email, PCAP HTTP analysis, audio spectrogram, Git logs | [View Writeup](forensics-stego/westbound.md) | `KPMG_CTF{g1t_r3m3mb3r5_wh4t_y0u_d3l3t3}` |
| **12** | **Vodoo Spirit of Death - 2**| PrivEsc / Linux | Easy | Sudo CVE-2021-3156 (Baron Samedit) heap overflow exploit | [View Writeup](privilege-escalation/vodoo-spirit-of-death-2.md) | `KPMG_CTF{AOnNtu2_K4uFWd5N0VlgTZL_A0Dl9D_reSvxZUOrOGQ}` |
| **13** | **Algebraic Nightmare** | Reverse Engineering | 30 pts | Custom VM bytecode (ANMBA2), dependency slicing, modular math | [View Writeup](reverse-engineering/algebraic-nightmare.md) | `KPMG_CTF{xUXOkGYKxH_0HVyXmwEKhUIodKep12X0dtHmzzhIEmk}` |
| **14** | **Logic Bypass** | Reverse Engineering | 20 pts | Unstripped ELF, `.rodata` key/ciphertext extraction, logic bypass | [View Writeup](reverse-engineering/logic-bypass.md) | `KPMG_CTF{2Ua-HX-GZrOszzeuq8gbRxXCUAoMQjxVTe2e-_PD4FY}` |
| **15** | **Granny Makes Food** | Web Exploitation | 20 pts | Decoy `/login` dead end, unlinked `POST /exploit` pickle RCE | [View Writeup](web/granny-makes-food.md) | Dynamic Per-Instance |
| **16** | **Arclight 1 & 2** | Web / GraphQL | 30 pts | Excessive Data Exposure, Nested Resolver BOLA (`staff.delegations`) | [View Writeup](web/arclight.md) | Multi-part |
| **17** | **Baud Intentions** | OT / IoT | 20 pts | SCADA UART web console, serial-derived maintenance key elevation | [View Writeup](ot-iot/baud-intentions.md) | `KPMG_CTF{aEX5aKaT2YDwmOronmiRKvr5lbUXHQpcwTtwF_d4lyM}` |
| **18** | **Latitude of Injection**| Android / SQLi | 20 pts | APK decompilation, geospatial coordinate lookup, SQLite SQL injection | [View Writeup](mobile-android/latitude-of-injection.md) | Completed |
| **19** | **OnboardX** | Web Exploitation | 30 pts | Mass Assignment, JWT RS256→HS256 confusion, ImageMagick MVG RCE | [View Writeup](web/onboardx.md) | `KPMG_CTF{aQUhfj8rvs240qLr7QZcMmhTRpRyssqXjvvgWRQv_zk}` |
| **20** | **Healthy Hijack - 1** | Privilege Escalation | 10 pts | Command runner prefix bypass via newlines, writable root script | [View Writeup](privilege-escalation/healthy-hijack-1.md) | `KPMG_CTF{9Uqut2wozejIRzFJkQKdnwydb7jOVDC51SvrAk-7T0I}` |
| **21** | **Healthy Hijack - 2** | Privilege Escalation | 10 pts | Command runner newline injection, root crontab script hijacking | [View Writeup](privilege-escalation/healthy-hijack-2.md) | `KPMG_CTF{q03a4OxG9_935IaRdz6pbmvezyD4IMZalQtwfdkNyJo}` |
| **22** | **Locks of Confusion - 1**| Web / Forensics | 15 pts | Archived provisioning logs, RSA private key modulus matching | [View Writeup](web/locks-of-confusion-1.md) | `KPMG_CTF{PKnI95kMvuXoqqaz_huqszzll_EK6RXa9aBQ4cmPOwQ}` |
| **23** | **Locks of Confusion - 2**| Web / SSH Escape | 15 pts | OpenSSH service identification, directory traversal, chroot privesc | [View Writeup](web/locks-of-confusion-2.md) | `KPMG_CTF{O6nl2A6ZmXtBVzWFeseaa_RGGkc1iqi0o0mL6PWd04c}` |
| **24** | **TradeBot** | Web / API Race | 20 pts | TOCTOU concurrency race condition, settlement engine oversubscription | [View Writeup](web/tradebot.md) | Sealed Record |

---

## 📂 Repository Directory Layout

```text
CTF_WRITEUPS/
├── README.md                          # Master navigation, challenge matrix & documentation
├── assets/
│   └── images/                        # High-resolution screenshots and proof-of-concept visual evidence
│       ├── guidelines-first-flag.png
│       ├── robots-welcome-flag.png
│       ├── onion-step*.png
│       ├── troll-clock-*.png
│       ├── vaultdocs*.png
│       └── securebot-*.png
├── web/                               # Web exploitation, GraphQL, API, and client-side challenges
│   ├── README.md
│   ├── arclight.md
│   ├── granny-makes-food.md
│   ├── locks-of-confusion-1.md
│   ├── locks-of-confusion-2.md
│   ├── onboardx.md
│   ├── tradebot.md
│   ├── vaultdocs-1.md
│   └── web-troll-clock.md
├── reverse-engineering/               # Compiled binary analysis, virtual machines, and logic bypass
│   ├── README.md
│   ├── algebraic-nightmare.md
│   ├── logic-bypass.md
│   └── secure-vault.md
├── forensics-stego/                   # Digital forensics, PCAP traffic analysis, audio & steganography
│   ├── README.md
│   ├── corrupted-vision.md
│   └── westbound.md
├── cryptography/                      # Multi-layer ciphers, encoding pipelines, and cryptanalysis
│   ├── README.md
│   └── onion.md
├── privilege-escalation/              # Linux host privilege escalation, kernel/sudo exploits & GTFOBins
│   ├── README.md
│   ├── healthy-hijack-1.md
│   ├── healthy-hijack-2.md
│   ├── vaultdocs-2.md
│   └── vodoo-spirit-of-death-2.md
├── ot-iot/                            # Operational Technology, SCADA, and embedded PLC interfaces
│   ├── README.md
│   └── baud-intentions.md
├── mobile-android/                    # Android application reversing (APK) & mobile SQLite injection
│   ├── README.md
│   └── latitude-of-injection.md
├── ai-security/                       # LLM prompt injection, jailbreaks, and excessive tool agency
│   ├── README.md
│   └── securebot-and-excessive-agency.md
└── misc/                              # Platform reconnaissance and rule verification
    ├── README.md
    └── welcome-and-guidelines.md
```

---

## 🛠️ Tooling & Methodology Summary

| Category | Primary Tools & Techniques |
| :--- | :--- |
| **Web Exploitation** | Burp Suite, Python `requests`, DevTools Console, OpenResty/Nginx auditing, JWT manipulation, GraphQL probing |
| **Reverse Engineering** | `file`, `readelf`, `llvm-objdump`, `nm`, `strings`, Ghidra, Python `struct`, backward dependency slicing |
| **Forensics & Stego** | Wireshark, `tshark`, Audacity (Spectrogram), OpenSSL, Git internals (`git show`), `xxd`, `file` |
| **Linux PrivEsc** | GTFOBins (`tar --checkpoint`), CVE-2021-3156 (Baron Samedit), crontab script hijacking, `sudo -l` auditing |
| **AI Security** | Persona adoption jailbreaks, JSON schema constraint injection, parameterized tool-call IDOR exploitation |
| **OT / IoT** | UART / Serial console terminal simulation, register extraction, hardware diagnostic unlock routines |

---

## 📌 Disclaimer
These writeups are created for educational and professional cybersecurity portfolio purposes, demonstrating offensive and defensive analysis methodologies. All exercises were conducted within authorized challenge environments.