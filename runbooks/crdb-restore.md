# CockroachDB Restore Runbook

Recover a CRDB cluster from the S3 backup chain produced by the
`cluster_backup` schedule. Pairs with `bootstrap_backup_schedule.py`
(creates the schedule) and `verify_backup.py` (5-minute freshness check
that pages on stale).

## When to use this runbook

- **Full cluster loss** — region wipe, accidental cluster decommission,
  catastrophic upgrade failure. Restore everything from the most recent
  full + incremental chain.
- **Accidental data wipe** — TRUNCATE / DROP TABLE / mass UPDATE run
  against the wrong target. Restore the affected table or database to a
  prior point in time.
- **App-bug PITR** — a deployed bug corrupted rows over a known window;
  restore to a timestamp just before the bug shipped.

A single dead node is **not** a restore scenario — replication
self-heals; replace the node via `replace_node` instead.

## Prerequisites

| Item | Where |
| -- | -- |
| Bucket name | Infisical CRDB project `da8253dd-…`, env `dev` or `prod`, key `BACKUP_S3_BUCKET` |
| CA cert | Infisical CRDB project, env-specific, key `CRDB_CA_CERT` |
| Root client cert | `bootstrap_backup_schedule` and `verify_backup` show the issuance flow; a one-off Windmill run can mint a 1h cert. For a hands-on session, copy the issuance block out of `verify_backup.py` |
| Target cluster | Existing (for table/database restore) **or** a freshly initialised cluster (for full-cluster restore — see [infra/docs/cockroachdb.md §Full Cluster Initialization (Fresh Start)](../../infra/docs/cockroachdb.md)) |
| SQL host | dev → `crdb.dev.overlay.solo.one`; prod → `db.prod.solo.one` |

All `RESTORE` statements below use `AUTH=implicit`, which makes CRDB sign
the S3 request with the cluster's instance role. The IAM policy is
applied automatically by `FcosWorkload` when `S3Config(enabled=True)`,
so prod and dev nodes can read from their own backup bucket without any
extra creds.

## List available backups

Connect as root (cert-based) and list the chain:

```sql
SHOW BACKUPS IN 's3://<bucket>/backups?AUTH=implicit';
```

This prints one row per full backup (one chain). To see the most recent
backup's start/end time and any incrementals attached to it:

```sql
SHOW BACKUP LATEST IN 's3://<bucket>/backups?AUTH=implicit';
```

## Scenario 1 — Restore latest (full cluster)

For total cluster loss. Bring up an empty cluster first via the
[Full Cluster Initialization (Fresh Start)](../../infra/docs/cockroachdb.md)
procedure, then:

```sql
RESTORE FROM LATEST IN 's3://<bucket>/backups?AUTH=implicit';
```

CRDB walks the chain (most recent full + every incremental appended to
it) automatically. Required: target cluster must be empty.

After the restore returns:

1. Verify row counts on the critical tables match the pre-incident
   counts (see _Post-restore validation_ below).
2. Re-run `provision_database` from Windmill to ensure the app's
   `network_app` user + grants are present. Full-cluster restore brings
   users along, but a re-run is idempotent and confirms the path is
   clean.
3. Update DNS (`db.{env}.solo.one`) to point at the new NLB if the
   restore targeted a new cluster.

## Scenario 2 — Point-in-time restore (full cluster)

Restore the whole cluster to a specific timestamp within the backup
window. `revision_history` on the schedule lets you target any second
in the chain, not just an incremental boundary.

```sql
RESTORE FROM '<full-backup-subdir>' IN 's3://<bucket>/backups?AUTH=implicit'
  AS OF SYSTEM TIME '2026-04-29 14:00:00';
```

`<full-backup-subdir>` is the subdirectory containing the full backup
that bounds your target timestamp (use `SHOW BACKUPS IN ...` to pick).

## Scenario 3 — Single-table or single-database restore

Live cluster, restore one table or database from the backup chain into
the existing cluster.

```sql
-- Database restore (into a new name to avoid clobbering the live one)
RESTORE DATABASE network FROM LATEST IN 's3://<bucket>/backups?AUTH=implicit'
  WITH new_db_name = 'network_restored';

-- Table restore (single table into an existing DB)
RESTORE TABLE network.public.consumer_consent
  FROM LATEST IN 's3://<bucket>/backups?AUTH=implicit'
  WITH into_db = 'network';
```

Use `network_restored` to inspect / swap rows, then drop it when done.
A direct table restore overwrites the live table — only run that if
you've confirmed the affected rows on a copy first.

## Post-restore validation

```sql
-- Row counts on critical tables. Compare against the last known-good.
SELECT 'entity', count(*) FROM network.public.entity UNION ALL
SELECT 'consumer_consent', count(*) FROM network.public.consumer_consent UNION ALL
SELECT 'product', count(*) FROM network.public.product;

-- Latest update timestamps (sanity check the PITR target).
SELECT max(updated_at) FROM network.public.entity;

-- Confirm app user + grants survived.
SHOW USERS;
SHOW GRANTS FOR network_app;
```

Then smoke test from the application side:

```bash
# From a network-api host on the same env
ssh solo@<network-api-overlay-ip>
sudo podman exec network-api curl -fsS http://localhost:8000/healthz
```

## Recovery scenarios (when not to use this runbook)

- **Single-node loss** — no restore needed. CRDB replicates every range
  3x by default; the replacement instance pulls missing ranges from
  surviving replicas. See `replace_node` Windmill script.
- **Regional outage** — out of scope. The cluster is single-region
  today. If we add multi-region, this runbook gets a new scenario.
- **App-data corruption while infra is healthy** — Scenario 3 first; do
  not pull the whole cluster.

## Rollback

A restore cannot itself be rolled back — once data is overwritten, the
prior live state is gone. Two mitigations to use **before** running
RESTORE in anger:

1. **Always restore to a new database name first** (`new_db_name=`)
   when working against a live cluster. Inspect the restored copy,
   verify expected rows, then swap with `ALTER DATABASE … RENAME` only
   after you're sure.
2. **For full-cluster restore**, point the restore at a fresh, empty
   cluster and cut DNS over only once smoke tests pass. Keep the old
   cluster around (read-only) until the new one is confirmed healthy.

## Alerting

`verify_backup` raises on `stale / stuck / failed / missing /
schedule_missing / schedule_paused`, which marks the Windmill run
failed. Two alerting paths fire on that failure:

1. **Slack** — `f/crdb/utils/slack.notify` posts a coloured attachment
   with the env tag and offending fields (job id, age, error). Reads
   the webhook URL from Windmill variable `f/crdb/SLACK_WEBHOOK_URL`.
   No-ops cleanly if the variable is unset.
2. **Elastic / Kibana** — Elastic Agent ships the failed run + the
   structured JSON log line (`backup_status: stale | stuck | …`).
   A Kibana alerting rule (operator config, not in this repo) pages
   on those.

To wire Slack: set the Slack incoming-webhook URL in the `crdb`
workspace variable `f/crdb/SLACK_WEBHOOK_URL`. The channel is
determined by the webhook itself.

## Related

- `bootstrap_backup_schedule.py` — registers the `cluster_backup`
  schedule (one-shot, idempotent).
- `verify_backup.py` — 5-minute freshness checker; a failed run is the
  alerting signal.
- `f/crdb/utils/slack.py` — shared Slack alerter, importable from any
  CRDB script via `from f.crdb.utils.slack import notify`.
- [infra/docs/cockroachdb.md](../../infra/docs/cockroachdb.md) —
  cluster bring-up, decommission, manual `BACKUP` / `RESTORE` syntax
  reference.
- [docs/runbooks/crdb-password-rotation.md](crdb-password-rotation.md)
  — sibling runbook for credential rotation.
