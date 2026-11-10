# pgaudit: writing audit records to a separate file or stream

## Question

Can pgaudit write its audit records to a separate file or stream, apart
from the regular PostgreSQL server log? If not, is there another option?

## Context

- This answer focuses on PostgreSQL on **VMs managed by Patroni** (the
  current platform). CloudNativePG is the future target and is covered only
  briefly.
- Audit records must not be lost. Keeping log volume down matters mainly to
  protect the log stream (see [limiting.md](limiting.md)).

## Answer

**pgaudit itself: no.** pgaudit writes its records through PostgreSQL's
normal logging, so they end up wherever `log_destination` points (`stderr`,
`csvlog`, `jsonlog`, `syslog`), mixed with all other server log messages.
pgaudit has no setting of its own for where its output goes.

Every audit record does start with a fixed prefix, which makes it easy to
separate the records later:

```
AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.t1,create table t1 (id int),<not logged>
AUDIT: OBJECT,2,1,READ,SELECT,TABLE,hr.salary,select * from hr.salary,<not logged>
```

There are three ways to get them into a separate file or stream.

### Option 1: pgauditlogtofile extension

[pgauditlogtofile](https://github.com/fmbiete/pgauditlogtofile) is a
separate extension. It sends pgaudit records to their own file instead of
the server log.

```ini
shared_preload_libraries = 'pgaudit,pgauditlogtofile'
pgaudit.log_directory     = '/var/log/postgresql/audit'
pgaudit.log_filename      = 'audit-%Y%m%d_%H%M.log'
pgaudit.log_rotation_age  = 1440        # minutes
pgaudit.log_rotation_size = 0           # 0 = disabled
pgaudit.log_connections    = on         # optional
pgaudit.log_disconnections = on         # optional
```

- **Pros:** a real separate file with its own rotation. The audit records
  leave the server log completely, which makes it smaller as well.
- **Cons:** a third-party extension with its own release cycle; the PGDG
  repositories package it (`pgauditlogtofile_<version>`). It has its own
  file format, so check that your log shipper can parse it. Check how it
  behaves when the audit file cannot be written (disk full, permissions)
  before relying on it for compliance.

### Option 2: syslog with rsyslog routing

```ini
log_destination = 'syslog'
syslog_facility = 'LOCAL0'
syslog_ident    = 'postgres'
```

```
# /etc/rsyslog.d/30-postgresql.conf
if $programname == 'postgres' and $msg contains 'AUDIT:' then {
    action(type="omfile" file="/var/log/postgresql/audit.log")
    stop
}
if $programname == 'postgres' then /var/log/postgresql/postgresql.log
```

This is the classic approach on VMs, but it has **serious drawbacks for
auditing**:

- **Messages can be lost.** The PostgreSQL documentation states that syslog
  can drop messages under load, whereas the logging collector never does.
  On systemd hosts, `/dev/log` usually goes through journald, whose rate
  limit (`RateLimitBurst`, `RateLimitIntervalSec`) silently drops messages.
  Use `syslog_sequence_numbers = on` (the default) to at least *detect*
  gaps.
- **Long records are split.** With `syslog_split_messages = on` (the
  default), messages longer than about 1 KB are split into several syslog
  messages. Only the first part contains `AUDIT:`, so the filter above
  sends the rest of a long statement to the wrong file. You can set
  `syslog_split_messages = off`, but then you have to raise rsyslog's
  maximum message size (`$MaxMessageSize`) as well.

### Option 3: logging collector plus a split in the log shipper (recommended)

Keep PostgreSQL writing to a local file through the logging collector, in a
structured format, and let the log shipper (Filebeat, Fluent Bit, Vector,
the OTel Collector, ...) send the audit records to a separate stream or
index:

```ini
logging_collector = on
log_destination   = 'jsonlog'                  # PostgreSQL 15+; else 'csvlog'
log_directory     = '/var/log/postgresql'      # outside PGDATA, see below
log_filename      = 'postgresql-%Y-%m-%d.log'
log_rotation_age  = '1d'
log_rotation_size = 0
```

In the shipper, route records whose `message` starts with `AUDIT:` to an
audit-only output with its own buffer, retention and priority.

- **Pros:** the logging collector is built to **never lose messages**:
  if it can't keep up, backends wait rather than drop records. A
  structured format (`jsonlog`/`csvlog`) keeps multi-line statements in one
  record, so the split is reliable. This is also the same model CNPG uses
  (JSON output, split in the pipeline), so the migration doesn't change how
  audit logs are handled.
- **Cons:** on disk, audit and server records share one file; the split
  happens only in the pipeline. If a separate file *on the host* is a hard
  requirement, use option 1.

## Patroni-specific notes

- **Keep `log_directory` outside PGDATA.** The default (`log`) is relative
  to the data directory. `patronictl reinit` and some rebuilds replace the
  data directory, and audit logs stored there are lost with it. Use an
  absolute path such as `/var/log/postgresql`.
- **Set logging parameters in the Patroni DCS configuration**
  (`patronictl edit-config`, under `postgresql.parameters`) so all nodes
  stay the same. Changing `logging_collector` or `shared_preload_libraries`
  requires a restart (`patronictl restart`); most other logging settings
  only need a reload.
- **Collect from every node.** After a failover or switchover, writes, and
  therefore audit records, come from another host. Replicas also produce
  audit records for read queries run against them.

## Recommendation

For Patroni on VMs:

1. **Use option 3:** `logging_collector = on`, `jsonlog` (or `csvlog`
   before PostgreSQL 15), `log_directory` outside PGDATA, and split on
   `AUDIT:` in the log shipper into a separate audit stream. This is the
   most reliable option, and it carries over directly to CNPG.
2. Use **pgauditlogtofile** only if a separate audit file on the host itself
   is required, for example by an auditor or a local SIEM agent.
3. **Avoid syslog** for audit records unless the drop and split issues
   above are addressed and monitored.

## CloudNativePG (outlook)

CNPG always logs to stdout as JSON and already tags pgaudit records as
`"logger": "pgaudit"`, with the audit fields parsed. Separating the audit
records is therefore done in the Kubernetes log pipeline, exactly like
option 3. pgauditlogtofile and syslog do not fit CNPG.

## References

- pgaudit: <https://github.com/pgaudit/pgaudit>
- pgauditlogtofile: <https://github.com/fmbiete/pgauditlogtofile>
- PostgreSQL, where to log (logging collector, syslog caveats):
  <https://www.postgresql.org/docs/current/runtime-config-logging.html>
- Patroni dynamic configuration:
  <https://patroni.readthedocs.io/en/latest/dynamic_configuration.html>
- CloudNativePG logging: <https://cloudnative-pg.io/documentation/current/logging/>
