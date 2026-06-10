# Publication Security and Privacy

## Scope

This document defines checks for publishing this readiness repository. Runtime
SBI security remains covered by
`14-transport-security-and-endpoint-trust.md`.

## Current Review Result

The tracked documents and reachable Git history were checked for common secret
and environment disclosures. No password, private key, SSH public key, API
key, OAuth token, bearer token, client secret, database credential, local
hostname, or absolute home-directory path was found.

The repository does contain Git author metadata, as every normal Git repository
does. New commits should use the configured GitHub `noreply` address. Existing
published commits retain their original author metadata unless history is
rewritten.

## Published-History Caveat

Earlier published document revisions contain internal task and numbered
implementation-stage labels that have now been removed from the working tree.
A normal follow-up commit updates the current branch view but does not erase
those terms from older commits.

The same applies to author email metadata from commits created before the
GitHub `noreply` address was configured. Rewriting and force-pushing public
history is a separate repository-maintenance decision and must not be done as
part of an ordinary documentation update.

## Public Examples

Network examples must use placeholders or documentation-only address ranges.
For IPv4 examples, prefer the RFC 5737 ranges:

- `192.0.2.0/24`;
- `198.51.100.0/24`;
- `203.0.113.0/24`.

Do not publish real laboratory addresses, NF instance identifiers, subscriber
identifiers, callback URIs, certificates, or configuration values merely to
make an example concrete.

## Internal Terminology

Public-facing prose should use standards-based feature names. Internal task
numbers, implementation-stage labels, team shorthand, and patch sequence names
should not appear unless a report is quoting historical evidence that cannot
be understood without the original term.

Exact branch names and commit IDs may remain where required for reproducible
Git analysis. Define their role once and use neutral terms such as "prototype
branch" in general discussion.

## Pre-Publication Gate

Before each push:

```text
[ ] Search tracked files and reachable history for credentials and private keys
[ ] Search for personal email addresses, usernames, hostnames, and home paths
[ ] Search for real testbed IP addresses, SUPIs, NF IDs, and callback URIs
[ ] Search for internal task, patch, and team shorthand
[ ] Confirm examples use placeholders or documentation address ranges
[ ] Confirm ignored reference material is not staged
[ ] Review git diff and staged files before committing
```

If a real secret is ever committed, deleting it in a later commit is not
sufficient. Revoke or rotate the secret first, then remove it from reachable
Git history before publishing the corrected repository.
