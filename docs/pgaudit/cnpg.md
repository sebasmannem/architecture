# pgaudit on CloudNativePG: what's different

## Question

What changes for pgaudit when moving from Patroni on VMs to
CloudNativePG (CNPG)?

## Context

- The other documents in this subject focus on Patroni on VMs. This page
  collects what is different on CNPG, so it can be used when planning the
  migration.
- Version details change quickly in CNPG. Check the points marked
  *(verify)* against the CNPG version you deploy.

## Answer

### 1. Enabling pgaudit

| | Patroni | CNPG |
|---|---|---|
| Packages | Install the PGDG pgaudit package on each VM | Included in the standard operand images. Minimal image variants don't include extensions *(verify the image you use)* |
| `shared_preload_libraries` | Add `pgaudit` yourself | **Added automatically** when any `pgaudit.*` parameter is set |
| `CREATE EXTENSION` | Run it yourself in each database | **Done automatically** by the operator in each database *(verify)* |
| Configuration | Patroni DCS (`patronictl edit-config`) | `spec.postgresql.parameters` in the `Cluster` resource |
| Restart | `patronictl restart` | Rolling restart by the operator, as configured with `primaryUpdateStrategy` |

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
spec:
  postgresql:
    parameters:
      pgaudit.log: "ddl, role"
      pgaudit.log_catalog: "off"
      pgaudit.log_parameter: "off"
      pgaudit.log_statement_once: "on"
      pgaudit.role: "auditor"
```

Removing all `pgaudit.*` parameters makes the operator unload pgaudit
again. That makes **who can edit the `Cluster` resource** an audit concern
(see point 4).

### 2. Per-database, per-role and object settings stay SQL

The cluster-wide settings are declarative, but these are not:

- `ALTER DATABASE ... SET pgaudit.log` and `ALTER ROLE ... SET pgaudit.log`;
- the object audit grants (`GRANT SELECT ON ... TO auditor`).

They are still done in SQL, for example in the application's migrations
or in a Kubernetes `Job`. The `auditor` role itself can be created
declaratively through CNPG's managed roles (`spec.managed.roles`, with
`login: false`). As far as I know, managed roles don't set per-role
parameters *(verify)*.

### 3. Logging is fixed by the operator

CNPG sets the logging parameters itself and doesn't let you override them:
the logging collector is always on and writes `csvlog`, which the instance
manager reads and writes to **stdout as JSON**. pgaudit records are
recognised and parsed into structured fields, with `"logger": "pgaudit"`.

Consequences compared to [separate-files.md](separate-files.md):

- **Separating audit records happens in the Kubernetes log pipeline**, by
  filtering on the `pgaudit` logger. That is the same model as the
  recommended option for Patroni.
- **pgauditlogtofile and syslog are not options.** They are not in the
  images, and files written inside the pod are lost with the pod.
- `log_line_prefix` doesn't apply, because the output is `csvlog`. The
  session ID and per-session line number are standard `csvlog` fields, so
  the loss detection from [log-stream.md](log-stream.md) still works.

### 4. Superuser and tampering move to Kubernetes

This is the biggest difference from the VM setup.

- **CNPG disables remote superuser access by default**
  (`enableSuperuserAccess: false`). The `postgres` user can then only
  connect from inside the pod.
- **Effective superuser = anyone allowed to `kubectl exec` into the
  database pods** (`pods/exec` in the namespace). They can switch auditing
  off for their session, and their activity shows up in pgaudit as the
  local `postgres` user, not as a person.
- **Anyone who can edit the `Cluster` resource** can change or remove the
  `pgaudit.*` parameters, or turn superuser access back on.
- **The Kubernetes API audit log** becomes an essential part of the audit
  trail: it records who ran `exec`, and who changed the `Cluster`
  resource. If the cluster resources are managed through GitOps, the Git
  history adds a reviewed record of configuration changes.

So the question "do app teams get superuser?" from
[added-value.md](added-value.md) becomes: **do app teams get `pods/exec`
or write access to `Cluster` resources in their namespace?** For audited
clusters, the answer should be no, or the exception must be documented.

### 5. The role model changes

- CNPG's default bootstrap creates an application database owned by an
  application user (`bootstrap.initdb.database` / `owner`). The current
  model, where the app team creates its own app role and database with a
  `CREATEDB`/`CREATEROLE` account, may be replaced by this, or by CNPG's
  declarative `Database` resources and managed roles.
- The audit conclusions from [added-value.md](added-value.md) still apply
  to whichever role owns the tables: an owner can still `REVOKE`
  privileges from `auditor`, so the SIEM alert on `GRANT`/`REVOKE` for the
  audit role stays necessary.

### 6. Logs are ephemeral, and Kubernetes rotates them quickly

This is the biggest risk for high-volume clusters.

- Container output is stored on the node in files that the kubelet rotates
  by size: by default **10 MB per file, 5 files**
  (`containerLogMaxSize`, `containerLogMaxFiles`). At the volumes from
  [log-stream.md](log-stream.md) (up to GB per minute), that's a few
  seconds of logs. If the shipper falls behind for longer than that,
  records are rotated away and lost.
- **When a pod is deleted or rescheduled, its container logs are deleted
  as well.** On VMs, log files stay on disk after a failover; on
  Kubernetes, they only survive if the shipper has already read them.
- Log volume counts against the node's ephemeral storage, and can lead to
  pods being evicted.

Mitigations:

- Raise `containerLogMaxSize` / `containerLogMaxFiles` on the nodes that run
  databases (a cluster-admin setting), or run high-volume databases on
  dedicated nodes with larger limits.
- Run the log shipper as a DaemonSet with position tracking and a disk
  buffer, and give it enough resources to keep up at peak volume.
- **Scope the auditing** ([limiting.md](limiting.md)) even more strictly
  than on VMs; there is much less buffer.
- **Detection matters even more**: the line-number gaps, session counts and
  heartbeat from [log-stream.md](log-stream.md) are how you find out that
  rotation or a pod deletion lost records. On CNPG, the session count can
  be collected with a custom query in CNPG's built-in Prometheus exporter,
  and the heartbeat can run as a Kubernetes `CronJob`.

### 7. Replicas and failover

As with Patroni, each instance (pod) produces its own audit records, and
replicas log read queries run against them. Collect logs from all
instances, and run the heartbeat against every instance, not only the
primary service.

## Summary of differences

| Topic | Patroni on VMs | CNPG |
|---|---|---|
| Enabling pgaudit | Manual: package, preload, extension | Automatic once `pgaudit.*` parameters are set |
| Where records go | Configurable: journal, syslog, files, pgauditlogtofile | Fixed: JSON on stdout, split in the Kubernetes log pipeline |
| Effective superuser | Database superusers and OS admins | Anyone with `pods/exec` or write access to the `Cluster` resource |
| Record of config changes | Patroni DCS history, pgaudit `MISC`/`ROLE` | Kubernetes API audit log, GitOps history |
| Log buffer | Log filesystem, sized by you | Kubelet-rotated files (default 50 MB per container) |
| Logs after failover | Stay on the VM | Lost with the pod unless already shipped |

## Recommendation

1. Use the same audit scope as on Patroni ([limiting.md](limiting.md)),
   set declaratively in the `Cluster` resource and managed through GitOps.
2. Treat `pods/exec` and write access to `Cluster` resources as superuser
   access: restrict them for audited clusters, and include the Kubernetes
   API audit log in the audit trail.
3. Keep per-role settings and object audit grants in migrations or a
   `Job`, and keep the SIEM alert on `GRANT`/`REVOKE` for the audit role.
4. Before moving high-volume teams, agree with the Kubernetes platform team
   on container log limits and log shipper capacity, and test the loss
   detection on CNPG.

## References

- CloudNativePG, PGAudit support:
  <https://cloudnative-pg.io/documentation/current/postgresql_conf/>
- CloudNativePG, logging: <https://cloudnative-pg.io/documentation/current/logging/>
- CloudNativePG, managed roles:
  <https://cloudnative-pg.io/documentation/current/declarative_role_management/>
- Kubernetes, container log rotation:
  <https://kubernetes.io/docs/concepts/cluster-administration/logging/>
- Kubernetes, API auditing:
  <https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/>
