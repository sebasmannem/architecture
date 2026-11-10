# pgaudit: keeping the log stream from losing audit records

## Question

How do we make sure the log stream can keep up, so that no audit records
are lost when there is a lot of activity, or when logging floods?

## Context

- PostgreSQL on VMs managed by Patroni. PostgreSQL output currently goes
  through Patroni into the **systemd journal**. There is no syslog or
  separate log shipping set up for PostgreSQL yet.
- The DBA team controls the PostgreSQL and Patroni configuration. The host
  (filesystems, journald, log shipper) is managed by the Linux team.
- No log loss has been observed yet, but PostgreSQL is known to keep
  running while logs are lost when the log path can't keep up.
- The team asking for auditing generates about **1 GB of WAL per minute**
  on a normal day, and may ask for every DML statement to be audited.

## Answer

### The basic trade-off

When audit records are produced faster than they can be written away,
there are only two possible outcomes:

1. **Drop records** and keep the database fast, or
2. **Slow the database down** (backpressure) until the records are written.

There is no third option. Which one you get depends on the log path:

| Log path | Under overload | Notes |
|---|---|---|
| **systemd journal** (current) | **Drops**, silently | journald rate-limits per service (by default about 10,000 messages per 30 seconds) and only writes a "Suppressed N messages" line. |
| **syslog** | **Drops** | The PostgreSQL docs say syslog prefers to drop messages when it can't keep up. Remote syslog over UDP drops even more easily. |
| **Logging collector** (`logging_collector = on`) | **Backpressure**: backends wait | The PostgreSQL docs say the logging collector is designed to never lose messages. Records are lost only if the log file can't be written at all (for example a full filesystem). |

So what you've seen, PostgreSQL carrying on while logs disappear, is
expected for the journal and syslog, which is the current setup. For audit
records, losing them silently is the worst outcome: an auditor can't tell
the difference between "nothing happened" and "we lost it". **For audit
data, choose backpressure, and then size everything so that backpressure
doesn't actually happen.**

### First priority: make loss detectable

No log path is guaranteed never to lose records. Even the logging collector
loses them when its filesystem is full, and the shipper, network and SIEM
can all fail further down the line. The real compliance risk is therefore
not loss itself, but **loss nobody notices**: then the whole audit trail
becomes untrustworthy, because "nothing happened" and "we lost it" look the
same. Loss that is detected, with the sessions and time window it affected,
can be explained in a comply-or-explain discussion.

The measures below detect loss **regardless of the log path**. They work
with the current journal setup.

1. **Number every log line per session.** `%l` in `log_line_prefix` gives
   each log line a per-session (or per-process) number starting at 1, and
   `%c` adds a unique session ID:

   ```ini
   log_line_prefix = '%m [%p] %c %l %q%u@%d '
   ```

   `jsonlog` and `csvlog` include both fields automatically
   (`session_id`, `line_num`). The number is assigned inside PostgreSQL,
   *before* the line enters the log path, so anything dropped afterwards
   leaves a gap. In the SIEM, a session with a gap in its line numbers
   (…41, 42, 45…) has **provably lost lines**.

2. **Log connections and disconnections** (`log_connections = on`,
   `log_disconnections = on`). Every session then has a known first and
   last line. A session without a disconnection record has lost its tail,
   or the server crashed, which is visible elsewhere.

3. **Count sessions as a cross-check.** `pg_stat_database.sessions`
   (PostgreSQL 14+) counts every session. Collect it with your metrics. If
   the SIEM saw fewer sessions than PostgreSQL counted for the same period,
   whole sessions were lost.

4. **Send a heartbeat from start to finish.** A scheduled job runs an
   audited statement every minute, for example a `SELECT` on a canary table
   covered by object logging (`GRANT SELECT ON audit.canary TO auditor`).
   The SIEM alerts if a heartbeat doesn't arrive in time. This catches
   what line numbering can't:
   - the pipeline is down;
   - auditing was switched off or reconfigured;
   - the journal or shipper has stalled.

   Run the heartbeat against every Patroni node, including replicas,
   because each node has its own log path.

5. **Use drop warnings as signals.** Alert on journald's
   `Suppressed N messages` lines, and on gaps in syslog sequence numbers
   (`syslog_sequence_numbers = on`) if syslog is ever used.

| Detected by | What it detects |
|---|---|
| Line-number gaps | Lines lost in the middle of a session |
| Missing disconnection record | Lines lost at the end of a session |
| Session count mismatch | Whole sessions lost |
| Missing heartbeat | Pipeline down, auditing disabled, delivery stalled |
| journald/syslog warnings | Drops in the transport, as reported by the transport itself |

**Who does what:** the PostgreSQL side (settings in Patroni, canary table,
heartbeat job) is entirely DBA work and can be done now. The gap,
session-count and heartbeat checks are SIEM rules, built by whoever runs
the SIEM.

**Verify before relying on it:** on a test cluster, force a drop (for
example by setting a very low journald rate limit) and check that it shows
up as a line-number gap and triggers the alert.

The measures in the rest of this document then **reduce how often** loss
happens. They no longer have to be trusted to prevent it.

### How much volume are we talking about?

A rough estimate for the 1 GB/minute team, if every DML statement is
audited:

- 1 GB WAL per minute is about 17 MB/s. With roughly 100–300 bytes of WAL
  per changed row, that's in the order of **50,000–170,000 row changes per
  second**.
- If those are mostly single-row statements, that's tens of thousands of
  audit records per second. At 300–800 bytes per record (`jsonlog` adds
  many fields), the audit log is **in the same order of magnitude as the
  WAL, or larger**: about 1–2 GB per minute, or 1.5–3 TB per day.
- With `pgaudit.log_parameter = on` it grows further. With multi-row
  statements or `COPY` it shrinks considerably.

This is an estimate. Measure the actual statement rate with
`pg_stat_statements`: run the query twice, a minute apart, and compare.

```sql
SELECT sum(calls) FILTER (WHERE query ~* '^\s*(insert|update|delete|merge)') AS dml_calls,
       sum(calls) FILTER (WHERE query ~* '^\s*(select|with)\M')            AS read_calls
FROM pg_stat_statements;
```

**At this volume, "audit every DML" is not an audit question any more.**
If the real need is a record of every data change, logical decoding / CDC
(for example Debezium to Kafka) is built for that throughput and also
records the data itself. pgaudit is better kept for DDL, roles, privileged
accounts and selected sensitive tables (see [limiting.md](limiting.md)).

### Design for Patroni on VMs

1. **Switch PostgreSQL to the logging collector.** This is configuration
   the DBA team controls (Patroni DCS, requires a restart):

   ```yaml
   postgresql:
     parameters:
       logging_collector: "on"
       log_destination: "jsonlog"            # PostgreSQL 15+; else csvlog
       log_directory: "/var/log/postgresql"  # outside PGDATA
       log_filename: "postgresql-%Y-%m-%d_%H%M.json"
       log_rotation_age: "1h"
       log_rotation_size: "1GB"
       log_truncate_on_rotation: "off"
   ```

   PostgreSQL's own output then no longer goes through the journal, so
   journald rate limiting stops affecting it. Patroni's own logs stay in
   the journal.

2. **Give the logs their own filesystem** (Linux team). A full log
   filesystem must not fill the root or data filesystem, and vice versa.
   Size it as a **buffer**: enough to hold several hours of peak volume, so
   a shipper or SIEM outage doesn't lead to loss. Monitor how full it is and
   alert well before it is full.

3. **Ship from the files with a shipper that tracks its position**
   (Filebeat, Fluent Bit, Vector, ...; Linux team). The shipper reads the
   files and remembers how far it got. If it falls behind, the files on
   disk are the buffer; nothing is lost while they are still there.
   - Split on `AUDIT:` into a **separate audit output** with its own
     (disk) buffer and priority, so the general server log can't crowd it
     out (see [separate-files.md](separate-files.md)).
   - Use at-least-once delivery. Duplicates are better than gaps.

4. **Only delete files that have been shipped.** Clean up with an
   age-based job whose age is safely larger than the maximum shipper lag.
   Don't use `logrotate` with `copytruncate` on PostgreSQL log files;
   PostgreSQL rotates its own files.

5. **Scope the auditing** as described in [limiting.md](limiting.md):
   no `READ`/`WRITE` session logging for the runtime role of this team,
   object logging only on the tables that need it,
   `pgaudit.log_parameter = off` and `pgaudit.log_statement_once = on`.

6. **Monitor the whole chain:**
   - log filesystem usage;
   - shipper lag (file offset vs. file size, or event age at the SIEM);
   - backend wait events: many sessions waiting on writing to the logging
     collector means it can't keep up and audit volume is slowing down the
     database;
   - changes to `pgaudit.*` and `logging_*` settings.

### If the journal stays for now

As a temporary measure, the Linux team can lift journald's rate limit for
the Patroni service (systemd 240 or later):

```ini
# /etc/systemd/system/patroni.service.d/logging.conf
[Service]
LogRateLimitIntervalSec=0
```

This removes the rate-limit drops, but the journal is still not a reliable
audit path: there is no guarantee against loss, and retention depends on
the journal's size limits. Treat it as a stopgap, not a solution.

### Even the logging collector has a limit

The logging collector is a single process. At very high record rates it
becomes a bottleneck, and then backends start waiting: the database slows
down instead of losing records. That is the right outcome for audit data,
but it means **scoping the auditing is not optional for a team at 1 GB of
WAL per minute**. It is what keeps auditing from becoming a performance
problem.

## Recommendation

1. **Make loss detectable first**, on the current journal setup: per-session
   line numbers in `log_line_prefix`, connection and disconnection logging,
   a session-count cross-check and a heartbeat on every node, with SIEM
   rules that alert on gaps and missing heartbeats. Test it with a forced
   drop.
2. **Measure** the statement rate before agreeing on what to audit. For
   "every DML" at this team's volume, propose **CDC** instead of pgaudit.
3. Lift the journald rate limit for Patroni as a quick win.
4. Move PostgreSQL logging from the journal to the **logging collector**,
   writing `jsonlog` to a dedicated filesystem outside PGDATA.
5. Have the Linux team set up a **file-based shipper** with position
   tracking, a separate audit output with a disk buffer, and cleanup only
   of shipped files.
6. **Monitor** filesystem usage, shipper lag and logging waits, and alert
   on them.
7. In the comply-or-explain documentation, describe how loss is detected
   and how detected loss is handled, rather than claiming it can't happen.

## CloudNativePG (outlook)

CNPG uses the logging collector internally and writes the result to stdout
as JSON. Kubernetes then stores container output in files that the kubelet
rotates by size (by default 10 MB, 5 files). At high volume, those files
can rotate away before the log shipper has read them, and records are
lost. This needs its own answer before a team at this volume moves to
CNPG.

## References

- PostgreSQL, where to log (logging collector vs. syslog):
  <https://www.postgresql.org/docs/current/runtime-config-logging.html>
- journald rate limiting: `man journald.conf`, `man systemd.exec`
  (`LogRateLimitIntervalSec`, `LogRateLimitBurst`)
- Patroni dynamic configuration:
  <https://patroni.readthedocs.io/en/latest/dynamic_configuration.html>
- Kubernetes container log rotation:
  <https://kubernetes.io/docs/concepts/cluster-administration/logging/>
