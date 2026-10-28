# Web Exploitation Writeups

This directory contains writeups for web applications, API security, and client-side challenges from KPMG CTF 2026 Campus Edition.

## Challenge Index

| Challenge | Points | Tech Stack | Vulnerability Classes | Final Flag |
| :--- | :--- | :--- | :--- | :--- |
| [Web Troll Clock](web-troll-clock.md) | 20 | WebSockets / JavaScript | Cosmetic client-side OTP bypass, DevTools fetch interception, timer sync | `KPMG_CTF{0U1gjUKP8iZUwkPUBq01yli5XZr7IFKUV2grCEWx4HY}` |
| [VaultDocs - 1](vaultdocs-1.md) | 10 | PHP / OpenResty / SSH | Local File Inclusion (LFI) via path traversal, `/etc/passwd` extraction, default SSH credentials (`vboxuser:changeme`) | `KPMG_CTF{2R68vP6s4_AUfTIldzjytnHuN76JGJn8MXL4IKy0R6k}` |
| [Granny Makes Food](granny-makes-food.md) | 20 | Flask / Python | Decoy `/login` rejection, unlinked `POST /exploit` route, Python `pickle.loads()` insecure deserialization RCE | Dynamic Flag |
| [Arclight 1 & 2](arclight.md) | 30 | GraphQL API | Excessive Data Exposure, Nested Resolver BOLA (`staff.delegations`), Broken Function-Level Authorization (`approveRequest`) | Multi-part |
| [OnboardX](onboardx.md) | 30 | Node.js / ImageMagick | Profile Mass Assignment (`PATCH /api/user/me`), JWT Algorithm Confusion (RS256 → HS256), ImageMagick MVG delegate RCE | `KPMG_CTF{aQUhfj8rvs240qLr7QZcMmhTRpRyssqXjvvgWRQv_zk}` |
| [Locks of Confusion - 1](locks-of-confusion-1.md) | 15 | Apache / Archival / RSA | Inventory reconnaissance, hidden HTML comments, `/verify` node check, RSA public modulus matching, ZIP decryption | `KPMG_CTF{PKnI95kMvuXoqqaz_huqszzll_EK6RXa9aBQ4cmPOwQ}` |
| [Locks of Confusion - 2](locks-of-confusion-2.md) | 15 | OpenSSH / Linux Chroot | Port identification (decoy vs OpenSSH), legacy SSH key authentication (`+ssh-rsa`), directory boundary escape, BusyBox `chroot` execution | `KPMG_CTF{O6nl2A6ZmXtBVzWFeseaa_RGGkc1iqi0o0mL6PWd04c}` |
| [TradeBot](tradebot.md) | 20 | JSON-RPC / API | Concurrency race condition, non-atomic capacity checks (TOCTOU), reserve vault oversubscription leak | Objective Sealed Record |
