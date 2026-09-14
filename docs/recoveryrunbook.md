# PostgreSQL Payment Service Recovery Runbook
## 1. Purpose
This runbook describes the recovery procedure for a PostgreSQL Payment Service database corruption incident.
The objective is to recover corrupted transactional data while:
* Minimizing data loss according to backup and WAL retention capability
* Minimizing service downtime (Low RTO)
* Preserving valid transactions
* Restoring healthy PostgreSQL Master-Slave replication
# 2. Incident Scenario
A production migration/script was executed incorrectly on the PostgreSQL Master database.
The incident caused:
* Incorrect DELETE operations
* Incorrect UPDATE operations
* Invalid or duplicated transaction records
Because Streaming Replication was enabledcorrupted WAL changes were replicated to the Slave database.
Important:
The Slave server cannot be considered a valid recovery source because it contains the same corrupted changes.
A PostgreSQL replica provides high availabilitybut it is not a replacement for backup.
# 3. Initial Incident Response
## Step 1 — Stop Application Writes
The first action is to prevent additional data changes.
Example:
bash
kubectl scale deployment payment-api \
-n payment-db \
--replicas=0
Verify:
bash
kubectl get pods -n payment-db
Purpose:
* Stop further corruption
* Preserve current database state
## Step 2 — Freeze Database Changes
Block unnecessary database operations.
Actions:
* Stop running migrations
* Disable automated deployment jobs
* Restrict database access
* Prevent manual production changes
## Step 3 — Collect Evidence
Collect Kubernetes and PostgreSQL evidence before recovery actions.
Check PostgreSQL logs:
bash
kubectl logs postgres-Master \
-n payment-db
Check pod status:
bash
kubectl describe pod postgres-Master \
-n payment-db
Check database activity:
sql
SELECT *
FROM pg_stat_activity;
Check replication status:
sql
SELECT
client_addr,
state,
sync_state,
sent_lsn,
write_lsn,
flush_lsn,
replay_lsn,
write_lag,
flush_lag,
replay_lag
FROM pg_stat_replication;
# 4. Identify Corrupted Data
The corruption window is identified from:
* PostgreSQL logs
* Application audit logs
* Database audit tables
* Transaction history
Example:
Incident Start:
12:45
Analyze affected records:
sql
SELECT *
FROM audit_logs
WHERE event_time >= '12:45';
Identify:
* Deleted transactions
* Modified transactions
* Invalid records
# 5. Pause Replication
Before recovery operationspause Slave replay.
Execute on PostgreSQL Slave only:
sql
SELECT pg_wal_replay_pause();
Verify:
sql
SELECT pg_is_wal_replay_paused();
Purpose:
* Prevent additional WAL replay during investigation
* Freeze Slave state for analysis
After recovery validation:
sql
SELECT pg_wal_replay_resume();
# 6. Create Emergency Backup
Before any repair operation:
Create a safety backup of the current database state.
Example:
bash
pg_basebackup \
-h postgres-Master \
-U replicator \
-D /backup/emergency \
-F tar \
-X stream
Purpose:
* Preserve current state
* Allow rollback if recovery operation fails
* Support incident investigation
# 7. Point In Time Recovery (PITR)
A temporary recovery database is created.
Example:
payment-recovery-db
Restore the database to a point before corruption:
Recovery Target:
12:44:59
Using pgBackRest:
bash
pgbackrest \
--stanza=payment \
--type=time \
--target="2026-09-14 12:44:59" \
restore
After reaching the recovery target:
* Validate recovered data
* Promote recovered instance for testing
# 8. Compare Production and Recovery Database
Two databases are compared:
Production:
payment_prod
Recovery:
payment_restore
## Find Deleted Transactions
sql
SELECT *
FROM payment_restore.transactions r
WHERE NOT EXISTS
(
SELECT 1
FROM payment_prod.transactions p
WHERE p.id = r.id
);
## Find Incorrect Updates
sql
SELECT
p.id,
p.amount,
p.status,
r.amount,
r.status
FROM payment_prod.transactions p
JOIN payment_restore.transactions r
ON p.id = r.id
WHERE
p.amount <> r.amount
OR p.status <> r.status;
Purpose:
* Identify corrupted records
* Preserve valid transactions created after the incident
# 9. Selective Data Repair
Only corrupted records are restored.
The full database should not be replaced because valid transactions after the incident time must be preserved.
## Restore Deleted Transactions
Example:
sql
INSERT INTO transactions
(
id,
user_id,
amount,
status,
created_at
)
SELECT
id,
user_id,
amount,
status,
created_at
FROM payment_restore.transactions
WHERE id IN
(
10001,
10002,
10003
);
## Restore Incorrect Updates
sql
UPDATE transactions t
SET
amount = r.amount,
status = r.status
FROM payment_restore.transactions r
WHERE t.id = r.id;
# 10. Rebuild PostgreSQL Slave Replica
Because the Slave database contains corrupted WAL changesrebuild the replica.
Stop Slave:
bash
systemctl stop postgresql
Preserve existing data directory:
bash
mv $PGDATA ${PGDATA}_old_$(date +%F_%H%M)
Clone from Master:
bash
pg_basebackup \
-h postgres-Master \
-D $PGDATA \
-U replicator \
-R
Start PostgreSQL:
bash
systemctl start postgresql
# 11. Validation
## Check Master
sql
SELECT count(*)
FROM transactions;
## Check Replica Recovery Mode
Execute on Slave:
sql
SELECT pg_is_in_recovery();
Expected:
true
## Check Replication Status
Execute on Master:
sql
SELECT
client_addr,
state,
sync_state
FROM pg_stat_replication;
Expected:
state = streaming
## Check Replication Lag
Execute on Slave:
sql
SELECT
now() - pg_last_xact_replay_timestamp();
# 12. Recovery Completion Checklist:
| Item                             | Status |
| -- |  |
| Application traffic restored     | ☐      |
| Corrupted records repaired       | ☐      |
| Transaction consistency verified | ☐      |
| Backup validated                 | ☐      |
| Replication healthy              | ☐      |
| Application smoke test completed | ☐      |
| Monitoring enabled               | ☐      |
| Incident documented              | ☐      |

