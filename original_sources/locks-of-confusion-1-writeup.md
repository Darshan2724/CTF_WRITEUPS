# Locks of Confusion - 1

## Challenge Information

- **Platform:** KICyber CTF
- **Challenge:** Locks of Confusion - 1
- **Category:** Web / Forensics
- **Target:** `http://kicyber-7974d4932921-locks-of-confusion1-0-0.chals.io`

## Flag

```text
KPMG_CTF{PKnI95kMvuXoqqaz_huqszzll_EK6RXa9aBQ4cmPOwQ}
```

## Challenge Description

The challenge presents an internal operations portal. The description says:

> Internal Ops just migrated an old database replica onto new hardware and swears everything was done "by the book". Somewhere sits a key that was decided long before it was ever generated.

The important clues are the old database replica, the migration, and the key that was carried forward from an earlier system.

## Reconnaissance

The portal exposes several operational pages:

```text
/
/inventory
/monitoring
/status
/tickets
/backups
/dns
/network
/cron-reports
/docs
/wiki/notes
```

The inventory identified a retired legacy host:

```text
legacy-03
Debian 4.0 (etch)
legacy DB read-replica
decommissioned -- retained for archival/compliance
```

The notes and migration documentation repeatedly referenced the `svc-legacydb`
account and a keypair originally created in 2007.

## Finding the Archive Paths

The page for `TICK-4471` contained HTML comments that were not visible in the
normal rendered page:

```html
<!-- audit follow-up: when cross-checking inventory for this incident, only
     the legacy-tier hosts are worth a second pass -- everything else in
     inventory already checks out. -->

<!-- TODO(infra): the tape-restore bundle from this incident never got
     cleaned up -- still sitting at
     /archive/provisioning/legacy-03-2007.tar.gz -->

<!-- TODO(infra): same for the old keyspace mirror -- clean this up too
     once the audit fully closes: /archive/keyspace/legacy-keys-2007.tar.gz -->
```

The two archive paths were therefore:

```text
/archive/provisioning/legacy-03-2007.tar.gz
/archive/keyspace/legacy-keys-2007.tar.gz
```

Initially, both returned:

```text
403 Forbidden -- complete the access check at /verify first.
```

## Completing the Access Check

The `/verify` page displayed an image containing several labeled network nodes
and instructed the solver to click a particular node. The required node and
its coordinates changed after reloads, so the label in the page had to be
read before clicking the matching node.

After selecting the correct node, the page reported:

```text
Verified. Archive downloads are unlocked for the next few minutes.
```

The archive URLs then returned successfully.

## Inspecting the Provisioning Archive

The provisioning archive contained files including:

```text
legacy-03-2007/home/svc-legacydb/.ssh/authorized_keys
legacy-03-2007/var/log/provision.log
legacy-03-2007/etc/motd
legacy-03-2007/etc/ssh/sshd_config.fragment
```

The provisioning log confirmed the historical key creation:

```text
Nov 14 2007 09:19:12 legacy-03 provision: ssh-keygen -t rsa -b 2048 -f /home/svc-legacydb/.ssh/id_rsa -N ''
Nov 14 2007 09:19:13 legacy-03 provision: public key installed to authorized_keys for svc-legacydb
```

The challenge description's statement that the key was "decided long before it
was ever generated" points to this old, carried-forward keypair rather than a
newly generated migration key.

## Matching the RSA Key

The keyspace archive contained 300 candidate private RSA keys:

```text
privkey_0000.pem
...
privkey_0299.pem
```

The public key in `authorized_keys` was compared with the public modulus from
each candidate private key. The matching candidate was:

```text
privkey_0137.pem
```

This matches the challenge's migration story: the original 2007 keypair was
carried through the 2011 P2V migration without regeneration.

## Discovering the Encrypted Backup

The keyspace archive also contained `README.txt`, which revealed another
archive:

```text
/archive/backups/e6f25638ae2ed6ba955cdae6a74c6798.zip
```

The README specified that the password was derived from the matching PEM file:

1. Use the matching private key.
2. Remove the `BEGIN` and `END` PEM lines.
3. Remove all line breaks.
4. Take the first 64 characters.

For `privkey_0137.pem`, the password was:

```text
MIIEowIBAAKCAQEAui2yyUzRtrahcJ1ZpfIrOBvT+s9ZI4k0mXQMegwGWfeKQ4cA
```

## Extracting the Flag

The encrypted ZIP contained:

```text
user.txt
```

After decrypting it with the derived password, the file contained:

```text
KPMG_CTF{PKnI95kMvuXoqqaz_huqszzll_EK6RXa9aBQ4cmPOwQ}
```

## Solution Chain

The complete chain was:

```text
Internal portal
  -> legacy-03 inventory entry
  -> TICK-4471 HTML comments
  -> /verify image check
  -> provisioning archive
  -> authorized_keys
  -> match against 300 private keys
  -> privkey_0137.pem
  -> first 64 PEM body characters
  -> encrypted ZIP
  -> user.txt
  -> flag
```
