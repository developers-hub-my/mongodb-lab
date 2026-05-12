# Security Hardening Checklist

The Module 3 hardening checklist, expanded with what each item means and
how to verify it. Use this when reviewing a MongoDB deployment at work.

## How to Use This Checklist

Walk through it once per environment (dev, staging, prod). Mark each item
Pass / Fail / N/A. Anything not Pass is a finding that needs a Jira ticket
or equivalent — not a "TODO later" note.

## Authentication and Access

### Authentication is enabled (`--auth` or `authorization: enabled`)

- **Why**: Without `--auth`, anyone who can reach port 27017 can read and
  modify every database.
- **Verify**: From the host, run `mongosh --host <ip>` with no credentials.
  You should be unable to read data. The command may connect, but
  `show dbs` should fail with `not authorized`.

### Strong, unique passwords (no defaults)

- **Why**: Default and weak passwords are the #1 root cause of public
  MongoDB compromises.
- **Verify**: No `ChangeMe`, `admin`, `password`, `mongo`, or vendor
  defaults. Length >= 16 characters. Stored in a secret manager, not env files.

### Application user uses a least-privilege role

- **Why**: A web app does not need to drop databases or manage users.
- **Verify**: `db.getUsers()` shows the app user has only `readWrite` (or
  narrower) on its own database. No `dbAdmin`, `userAdmin`, or `root` roles.

### Separate admin and application accounts

- **Why**: Admin operations and app operations have different blast radius.
- **Verify**: Inspect connection strings used by deployed apps. None
  should be using `admin` or `root`.

## Encryption and Network

### TLS enabled for all connections

- **Why**: Without TLS, passwords and document contents travel in plaintext
  on the wire — visible to anyone on the network path.
- **Verify**: MongoDB config has `net.tls.mode: requireTLS` and a valid
  certificate chain. Client connections use `tls=true` in the URI.

### Encryption at rest (or filesystem-level)

- **Why**: A stolen disk image otherwise reveals every document.
- **Verify**: Either MongoDB Enterprise's encrypted storage engine is
  configured, or the underlying volume is encrypted (LUKS, EBS encryption,
  similar).

### Bind to specific interfaces, not `0.0.0.0`

- **Why**: `bindIp: 0.0.0.0` listens on every interface, including public
  ones if any are attached.
- **Verify**: `net.bindIp` in `mongod.conf` is a list of trusted
  interfaces (often `127.0.0.1,<private-ip>`). Confirm with
  `ss -tlnp | grep 27017` on the host.

### Firewall protects port 27017

- **Why**: Defense in depth — even if `bindIp` widens by mistake, a
  firewall still blocks unauthorized access.
- **Verify**: Security group, `iptables`, `nftables`, or cloud firewall
  rule denies 27017 from anywhere except the application subnet.

## Operations

### Daily automated backups, tested monthly

- **Why**: A backup that has never been restored is not a backup.
- **Verify**: There is a scheduled `mongodump` (or equivalent) running
  daily. There is a runbook to restore one and a log entry showing it
  was last executed within 30 days.

### Backups encrypted and access-controlled

- **Why**: Backups contain the same data as the database — they need the
  same protection.
- **Verify**: Backup storage uses encryption at rest. Access is granted
  via IAM/RBAC to a small group, audit-logged.

### Audit logs shipped to SIEM

- **Why**: Detection only works if someone is looking at the logs.
- **Verify**: MongoDB audit logs (Enterprise) or change-stream events
  flow to your central log platform. There are alerts on
  privilege grants, role changes, and failed authentications.

### Patch cycle: track MongoDB CVEs

- **Why**: Older MongoDB versions accumulate published CVEs.
- **Verify**: There is an owner for the MongoDB patch cycle. Production
  is within N minor versions of upstream. CVE feed is monitored.

## Quick Self-Score

Count items marked **Pass**:

| Score | Status |
|-------|--------|
| 11 / 11 | Strong baseline — keep it audited |
| 8-10 / 11 | Acceptable for non-sensitive data — close the gaps |
| 5-7 / 11 | Borderline — start a remediation sprint |
| <= 4 / 11 | Treat as an incident-in-waiting |

## Related Documentation

- [Exercise 3 — Security Hardening](../02-exercises/03-security-hardening.md)
- [mongosh Cheatsheet](01-mongosh-cheatsheet.md) — user-management commands
- [MongoDB Security Manual](https://www.mongodb.com/docs/manual/security/) (upstream)
