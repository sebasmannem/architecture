# pgaudit

Architectural questions about auditing PostgreSQL with
[pgaudit](https://github.com/pgaudit/pgaudit).

## Context

- PostgreSQL runs on VMs managed by **Patroni**, with a migration to
  **CloudNativePG** planned. The answers focus on Patroni, with notes on
  CNPG where relevant.
- Compliance requirements apply on a **comply or explain** basis, and are
  currently met.
- **Superuser** access is held by the DBA team, and in some cases by app
  teams.
- **Role model:** the DBA team gives each app team an account with
  `CREATEDB` and `CREATEROLE`. With it, the app team creates an app role
  that creates the database and grants access. Applications run with a
  least-privileged runtime role, mostly through PgBouncer. A separate
  migration role may be used for schema changes.
- PostgreSQL logs currently go through Patroni into the **systemd journal**.
  Some teams generate a lot of activity (around 1 GB of WAL per minute).

## Questions

| # | Question | Answer |
|---|---|---|
| 1 | Can pgaudit limit logging, for example per table or per statement type (DDL, DML, ...)? | [limiting.md](limiting.md) |
| 2 | Can pgaudit write audit records to a separate file or stream? | [separate-files.md](separate-files.md) |
| 3 | What does pgaudit add, and what doesn't it do? | [added-value.md](added-value.md) |
| 4 | How do we keep the log stream from losing audit records? | [log-stream.md](log-stream.md) |

## Key conclusions

1. **Scope the auditing.** Log `DDL` and `ROLE` cluster-wide, log more for
   personal and admin accounts, and use object logging only for sensitive
   tables. Avoid `READ`/`WRITE` session logging for application roles.
   pgaudit can't audit DDL per table; use event triggers if that's needed.
   ([limiting.md](limiting.md))
2. **pgaudit can't write to its own file or stream.** Use the logging
   collector with `jsonlog` and split on `AUDIT:` in the log shipper. Use
   pgauditlogtofile only if a separate file on the host is required, and
   avoid syslog for audit records. ([separate-files.md](separate-files.md))
3. **pgaudit is a source of audit records, not an audit-proof solution.**
   It doesn't protect against superusers (or, before PostgreSQL 16,
   `CREATEROLE` holders), doesn't log logins, data values or activity
   outside SQL, and can't identify end users behind shared accounts.
   Those gaps need other controls. ([added-value.md](added-value.md))
4. **Make loss detectable first, then reduce it.** Per-session line
   numbers, connection logging, session counts and a heartbeat reveal lost
   records on any log path. The logging collector, a dedicated filesystem
   and a file-based shipper reduce how often loss happens. For "audit every
   DML" at high volume, CDC is a better fit than pgaudit.
   ([log-stream.md](log-stream.md))

## Open points

- CloudNativePG logging at high volume (Kubernetes log rotation can
  discard records before they're shipped) needs its own answer.
