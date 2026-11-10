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

## In our environment

### Role model

| Role | Created by | Privileges | Used for |
|---|---|---|---|
| App team account | DBA team | `CREATEDB`, `CREATEROLE` (can create roles and grant to them) | App team creates the app role |
| App role | App team | `CREATEDB`; owns the database and sets its permissions | Creates the database and grants access |
| Migration role (optional) | App team | Can change the schema | Schema migrations, e.g. at application start-up |
| Runtime role | App team | Read/write only | The application, mostly through PgBouncer |
| Superuser | DBA team, **in some cases also the app team** | Everything | Administration |

### What this means for auditing

1. **Superuser access by app teams makes pgaudit unreliable for those
   teams.** A superuser can switch auditing off for its own session or for
   the whole cluster, and can run OS commands as the `postgres` user
   (`COPY ... TO PROGRAM`), which includes editing local log files. For
   clusters where an app team has superuser, pgaudit only shows what that
   team *chooses* to leave visible. This must be stated explicitly in the
   comply-or-explain documentation. The default should be **no superuser
   for app teams**, with exceptions documented per cluster.

2. **`CREATEROLE` is close to superuser before PostgreSQL 16.** Up to
   PostgreSQL 15, a role with `CREATEROLE` can grant itself or other roles
   membership in almost any non-superuser role. That includes
   `pg_execute_server_program`, which allows OS commands, with the same
   consequences as point 1. PostgreSQL 16 changed this: a `CREATEROLE` role
   then only manages roles it created itself. **On clusters older than
   PostgreSQL 16, treat app team accounts as near-superuser for audit
   purposes.**

3. **App teams cannot lower the audit settings.** `pgaudit.*` settings are
   superuser-only, so `CREATEROLE` is not enough to run
   `ALTER ROLE ... SET pgaudit.log = 'none'` (unless someone has explicitly
   run `GRANT SET ON PARAMETER`, which should not be done for `pgaudit.*`).

4. **Schema migrations are covered by cluster-wide DDL logging.** App teams
   create their own migration roles, so the DBA team doesn't know their
   names and can't set per-role audit settings on them. That's fine:
   `pgaudit.log = 'ddl, role'` cluster-wide (see
   [limiting.md](limiting.md)) captures every migration, whichever role
   runs it. Role creation and grants by the app team account and app role are captured by
   the `ROLE` class.

5. **App teams can switch off object auditing on their own tables.** The
   app team owns the tables, and a table owner can `REVOKE` privileges from
   the `auditor` role, silently ending object auditing for that table.
   The `REVOKE` itself is logged (class `ROLE`), so add a **SIEM alert on
   `GRANT`/`REVOKE` statements involving the audit role**.

6. **Runtime activity shows the runtime role and the PgBouncer host, not a
   person.** For the runtime role, pgaudit answers "did the application do
   this?", not "who did it?". End-user accountability has to come from the
   application's own audit trail. Migrations run from a CI/CD pipeline are
   traceable to the pipeline run, not to a person.

7. **The split between migration and runtime roles helps.** Schema changes
   come from one role and data access from another, which makes the audit
   records easier to interpret and to alert on, for example DDL from the
   runtime role is always suspicious.

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
