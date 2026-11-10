# pgaudit: limiting what gets logged

## Question

Can pgaudit limit audit logging, for example per table or per type of
statement (DDL, DML, ...)? If not, is there another option?

## Context

- PostgreSQL runs on VMs managed by Patroni, with a migration to
  CloudNativePG (CNPG) planned.
- Compliance requirements apply on a *comply or explain* basis. We currently
  comply.
- We want to keep log volume down, mainly to protect the log stream: if the
  stream becomes overloaded, audit records can be lost. (Protecting the
  stream itself is covered in a separate question.)

## Answer

Partly. pgaudit can filter by **statement class** (and by database or role),
and it can filter **per table**, but only for `SELECT`, `INSERT`, `UPDATE`
and `DELETE`. It cannot filter DDL per table.

pgaudit has two modes that can be combined.

### 1. Session audit logging (`pgaudit.log`)

Logs statements by **class**:

| Class      | Covers                                                    |
|------------|-----------------------------------------------------------|
| `READ`     | `SELECT`, `COPY ... TO`                                   |
| `WRITE`    | `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `COPY ... FROM` |
| `FUNCTION` | Function calls and `DO` blocks                            |
| `ROLE`     | `GRANT`, `REVOKE`, `CREATE/ALTER/DROP ROLE`               |
| `DDL`      | All DDL not covered by `ROLE`                             |
| `MISC`     | `DISCARD`, `FETCH`, `CHECKPOINT`, `VACUUM`, `SET`         |
| `MISC_SET` | `SET ROLE` and other `SET` commands                       |
| `ALL`      | Everything above                                          |

You can exclude a class with a minus sign, for example
`pgaudit.log = 'all, -misc'`.

This can be set at several levels:

```sql
-- cluster-wide (postgresql.conf / Patroni DCS / CNPG spec)
pgaudit.log = 'ddl, role'

-- per database
ALTER DATABASE app SET pgaudit.log = 'ddl, role';

-- per role: e.g. log everything for human DBA accounts
ALTER ROLE dba_jan SET pgaudit.log = 'all, -misc';
```

All pgaudit settings are superuser-only, so ordinary users cannot switch off
their own auditing. Session logging has **no per-table filter**.

### 2. Object audit logging (`pgaudit.role`)

This is pgaudit's per-table (and even per-column) mechanism. You name an
audit role, and pgaudit logs any statement that uses a privilege granted
to that role:

```sql
CREATE ROLE auditor NOLOGIN;
-- pgaudit.role = 'auditor'  (set in configuration)

GRANT SELECT, UPDATE ON hr.salary TO auditor;     -- whole table
GRANT SELECT (bsn) ON crm.customer TO auditor;    -- one column only
```

These statements are then logged with audit type `OBJECT`.

Limitations:

- Only `SELECT`, `INSERT`, `UPDATE` and `DELETE` are covered. There is
  **no per-table DDL auditing**.
- There is only one audit role. It can be granted membership of other roles
  to group grants, though.
- The grants have to be maintained, for example in schema migrations, or new
  sensitive tables go unaudited.

### Settings that reduce volume

| Setting                             | Effect                                                                      |
|-------------------------------------|-----------------------------------------------------------------------------|
| `pgaudit.log_catalog = off`         | Skips statements that only touch `pg_catalog` (psql, GUI tools, ORMs). Large reduction in noise. |
| `pgaudit.log_parameter = off`       | Leaves bind parameters out (default). Smaller records, and keeps sensitive values out of the logs. |
| `pgaudit.log_statement_once = on`   | Logs the statement text only once per statement, not once per sub-statement. |
| `pgaudit.log_relation`              | With `on`, logs a separate entry for each relation in `READ`/`WRITE` statements. Leave it `off` unless you need it. |
| `pgaudit.log_level`                 | Sets the log level of audit entries (only relevant when `log_min_messages` would filter them). |

### Other options for what pgaudit cannot do

| Need                                   | Alternative                                                                   | Caveats                                                                    |
|----------------------------------------|-------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| DDL per table/schema                   | Event triggers on `ddl_command_end` / `sql_drop`, filtering on object identity, writing to a table or `RAISE LOG` | Do not fire for global objects (roles, databases, tablespaces); must be maintained |
| DML per table *including data* (old/new values) | Row-level audit triggers (e.g. the classic `audit-trigger`, pgMemento) | No `SELECT` capture; write overhead; a table owner can disable them |
| History of data changes outside the database | Logical decoding / CDC (e.g. Debezium)                                  | Records *what* changed, not *who ran which statement*                      |
| Very coarse filtering                  | `log_statement = ddl` / `mod`                                                 | No per-table filter; plain log lines with no audit structure               |

## Recommendation

1. **Session logging broadly, for low-volume classes:**
   `pgaudit.log = 'ddl, role'` cluster-wide, with `pgaudit.log_catalog = off`.
   Add `misc_set` if `SET ROLE` needs to be traceable.
2. **More detail for personal and admin accounts:**
   `ALTER ROLE <dba> SET pgaudit.log = 'all, -misc'`. Application accounts
   produce most of the volume; people produce most of the risk.
3. **Object logging only for sensitive tables:** grant `SELECT`/`WRITE`
   privileges to `auditor` on the tables or columns with personal or
   financial data. Manage these grants in the schema migrations, next to the
   tables they cover.
4. **Avoid `READ`/`WRITE` session logging for application roles.** It
   produces the most volume and is the biggest threat to the log stream.
5. Use **event triggers** only if per-object DDL auditing turns out to be
   an actual requirement.

## Platform notes

### Patroni (current)

- Configure the settings in the Patroni DCS config
  (`postgresql.parameters`). Adding `pgaudit` to `shared_preload_libraries`
  requires a restart (`patronictl restart`). The `pgaudit.*` settings only
  need a reload.
- Run `CREATE EXTENSION pgaudit;` in each database.

### CloudNativePG (target)

- pgaudit is included in the standard CNPG operand images.
- Set the parameters in `spec.postgresql.parameters`. When CNPG sees
  `pgaudit.*` parameters, it adds `pgaudit` to `shared_preload_libraries`
  and creates the extension itself.
- Per-database and per-role settings (`ALTER DATABASE/ROLE ... SET`) and the
  object audit grants are still done in SQL (or migrations). Keep that in
  mind when moving the setup to the Kubernetes manifests.
- CNPG logs to stdout as JSON and parses pgaudit entries into structured
  fields. See the log-stream question for what that means for volume.

```yaml
spec:
  postgresql:
    parameters:
      pgaudit.log: "ddl, role"
      pgaudit.log_catalog: "off"
      pgaudit.log_parameter: "off"
      pgaudit.role: "auditor"
```

## References

- pgaudit README: <https://github.com/pgaudit/pgaudit>
- PostgreSQL event triggers: <https://www.postgresql.org/docs/current/event-triggers.html>
- CloudNativePG, PostgreSQL configuration / pgaudit:
  <https://cloudnative-pg.io/documentation/current/postgresql_conf/>
- CloudNativePG, logging: <https://cloudnative-pg.io/documentation/current/logging/>
