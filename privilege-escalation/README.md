# Linux Privilege Escalation Writeups

This directory contains writeups for Linux host privilege escalation, container escapes, and local exploit challenges from KPMG CTF 2026 Campus Edition.

## Challenge Index

| Challenge | Points | Starting Context | Escalation Vector | Final Flag |
| :--- | :--- | :--- | :--- | :--- |
| [Vodoo Spirit of Death - 2](vodoo-spirit-of-death-2.md) | Easy | `maya.chen` SSH | CVE-2021-3156 (Baron Samedit Sudo Heap Overflow) to Root Shell | `KPMG_CTF{AOnNtu2_K4uFWd5N0VlgTZL_A0Dl9D_reSvxZUOrOGQ}` |
| [Healthy Hijack - 1](healthy-hijack-1.md) | 10 | Web Command Runner | Incomplete prefix allowlist newline injection & writable `/opt/healthcheck.sh` | `KPMG_CTF{9Uqut2wozejIRzFJkQKdnwydb7jOVDC51SvrAk-7T0I}` |
| [Healthy Hijack - 2](healthy-hijack-2.md) | 10 | Web Command Runner | Isolated command subshell injection & root crontab script hijacking | `KPMG_CTF{q03a4OxG9_935IaRdz6pbmvezyD4IMZalQtwfdkNyJo}` |
| [VaultDocs - 2](vaultdocs-2.md) | 10 | `vboxuser` SSH | Sudoers `/bin/tar` GTFOBins checkpoint command execution to Root | `KPMG_CTF{vujzSbEPjuGNy7fw7JMzFwgXuk21JyU8zB2fIaDqGmg}` |
