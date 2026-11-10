# pgaudit: what it adds, and what it doesn't

## Question

What is the added value of pgaudit, and what isn't?

## Context

- PostgreSQL on VMs managed by Patroni, with a migration to CloudNativePG
  planned.
- Compliance requirements apply on a *comply or explain* basis.
- Related answers: [limiting.md](limiting.md) (what to log) and
  [separate-files.md](separate-files.md) (where the records go).

## What pgaudit adds

### 1. It logs what actually ran, not just what was sent

`log_statement` logs the text the client sent. pgaudit hooks into the
executor and utility processing, so it logs what PostgreSQL actually did:

```sql
DO $$
BEGIN
  EXECUTE 'DROP TABLE ' || 'hr.salary';
END $$;
```

- `log_statement = 'ddl'` logs nothing: a `DO` block is not DDL. With
  `log_statement = 'all'` you see only the `DO` block.
- pgaudit logs the `DO` block **and** a separate record for
  `DROP TABLE hr.salary`.

The same applies to SQL inside functions: tables read or changed by a
function are logged as sub-statements.

### 2. Records are structured and easy to filter

Every record has a fixed layout:

```
AUDIT: <type>,<statement id>,<substatement id>,<class>,<command>,<object type>,<object name>,<statement>,<parameters>
AUDIT: SESSION,12,1,DDL,ALTER TABLE,TABLE,hr.salary,alter table hr.salary add column bonus int,<not logged>
```

A SIEM can filter on, for example, "all DDL on schema `hr`" or "all `ROLE`
statements" without parsing SQL.

### 3. It can audit reads of specific tables and columns

Object audit logging (`pgaudit.role`) is the only practical way in
PostgreSQL to record *who read which sensitive table or column*. Triggers
can't capture `SELECT`, and `log_statement` can't filter per object. See
[limiting.md](limiting.md).

### 4. Ordinary users can't switch it off

All `pgaudit.*` settings are superuser-only. A regular user or application
account cannot `SET pgaudit.log = 'none'` to hide its own activity.

### 5. It's the recognised standard

The CIS Benchmark for PostgreSQL recommends pgaudit, and it's available
nearly everywhere: PGDG packages, CloudNativePG images, and managed
services such as Amazon RDS and Azure Database for PostgreSQL. In a
comply-or-explain discussion, "we use pgaudit" is easy to explain; a
home-made solution needs much more justification.

## What pgaudit doesn't do

| Gap | Why it matters | Covered by |
|---|---|---|
| **Not tamper-proof against superusers or OS admins.** A superuser can turn auditing off (`ALTER SYSTEM`, `SET`), and someone with OS access can edit or delete log files. | pgaudit on its own does not make audit records trustworthy. | Ship logs off the host quickly; keep DBA and log/SIEM admin roles separate; minimise superuser accounts; alert on changes to `pgaudit.*` settings. |
| **No connection or login auditing.** | Who logged in, from where, and failed logins are typical audit requirements. | `log_connections = on`, `log_disconnections = on`. |
| **No data values or before/after images.** Only bind parameters, and only with `pgaudit.log_parameter = on`. | It is not a change history: you see *that* a row was updated, not what the old value was. | Audit triggers or CDC (change data capture), see [limiting.md](limiting.md). |
| **Statements that fail to parse aren't logged.** Errors only appear in the normal server log. | Failed or probing attempts can be missed if only `AUDIT:` records are collected. | `log_min_error_statement = error`, and send the server log (not only the audit stream) to the SIEM. |
| **No end-user identity behind shared application accounts or a connection pooler.** | Records show `app_user` connecting from the PgBouncer host, not the person or client behind it. | Personal accounts for people; the application sets `application_name` (included in `log_line_prefix`) or keeps its own audit trail. |
| **Sees nothing outside SQL**: `pg_basebackup`, pgBackRest, physical replication, file access on the host. | A full copy of the data can be taken without any pgaudit record. | Access control and logging at OS, backup and network level; restrict replication and backup roles. |
| **No alerting or analysis.** | It only writes records; nobody is notified when something suspicious happens. | SIEM rules and alerting. |
| **Adds volume and some overhead.** | Broad `READ`/`WRITE` session logging can overload the log stream and slow down busy workloads. | Scope it as described in [limiting.md](limiting.md). |

## When pgaudit isn't worth it

- **Clusters with no sensitive data and no audit requirement** (for example
  test environments with synthetic data): `log_statement = 'ddl'` plus
  connection logging is usually enough.
- **When the real requirement is a data change history** ("what was the
  value last month?"): use triggers or CDC instead; pgaudit doesn't answer
  that question.

For production clusters with personal or business-critical data, the
cost of pgaudit with a limited scope is low, and it is hard to explain
*not* having it.

## Recommendation

1. Use pgaudit as the **source** of SQL audit records, scoped as described
   in [limiting.md](limiting.md).
2. Treat the gaps above as part of the same design. An audit-proof setup
   also needs:
   - off-host shipping and separation of duties
     (see [separate-files.md](separate-files.md));
   - `log_connections` / `log_disconnections`;
   - the error log in the SIEM;
   - few superusers, and personal accounts for people;
   - OS, backup and replication access controlled and logged separately.
3. In a comply-or-explain discussion, document which gaps are covered by
   which control, using the table above as a checklist.

## References

- pgaudit: <https://github.com/pgaudit/pgaudit>
- PostgreSQL error reporting and logging:
  <https://www.postgresql.org/docs/current/runtime-config-logging.html>
- CIS Benchmarks (PostgreSQL): <https://www.cisecurity.org/benchmark/postgresql>
