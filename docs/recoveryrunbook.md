# PostgreSQL Payment Service Recovery Runbook
## 1. Purpose
This runbook describes the recovery procedure for a PostgreSQL Payment Service database corruption incident.
The objective is to recover corrupted transactional data while:
* Minimizing data loss (RPO close to zero)
* Minimizing service downtime (Low RTO)
* Preserving valid transactions
* Restoring healthy Master-Slave replication
# 2. Incident Scenario
A production migration/script was executed incorrectly on the PostgreSQL Master database.
The incident caused:
* Incorrect DELETE operations
* Incorrect UPDATE operations
* Invalid or duplicated transaction records
Because Streaming Replication was enabled, corrupted WAL changes were replicated to the standby database.
Important:
The standby server cannot be considered a valid recovery source because it contains the same corrupted changes.
# 3. Initial Incident Response
## Step 1 — Stop Application Writes
The first action is to prevent additional data changes.
Example:
```bash
kubectl scale deployment payment-api --replicas=0
```
or redirect traffic to maintenance mode.
Purpose:
* Stop further corruption
* Preserve current database state
## Step 2 — Freeze Database Changes
Block unnecessary database operations.
Actions:
* Stop running migrations
* Disable automated deployment jobs
* Restrict database access
## Step 3 — Collect Evidence
Collect:
PostgreSQL logs:
```bash
kubectl logs postgres-master
```
Database activity:
```sql
SELECT *
FROM pg_stat_activity;
```
Replication status:
```sql
SELECT
client_addr,
state,
sent_lsn,
write_lsn,
flush_lsn,
replay_lsn
FROM pg_stat_replication;
```
# 4. Identify Corrupted Data
The corruption window is identified from logs.
Example:
```
Incident Start:
12:45
```
Analyze affected records:
```sql
SELECT *
FROM audit_logs
WHERE event_time >= '12:45';
```
Identify:
* Deleted transactions
* Modified transactions
* Invalid records
# 5. Pause Replication
Before recovery operations, pause standby replay.
On PostgreSQL standby:
```sql
SELECT pg_wal_replay_pause();
```
Verify:
```sql
SELECT pg_is_wal_replay_paused();
```
Purpose:
Prevent additional WAL replay during investigation.
# 6. Create Emergency Backup
Before any repair operation:
Create a safety backup.
Example:
```bash
pg_basebackup \
-h postgres-master \
-U replicator \
-D /backup/emergency \
-F tar \
-X stream
```
Purpose:
Allow rollback if recovery operation fails.
# 7. Point In Time Recovery (PITR)
A temporary recovery database is created.
Example:
```
payment-recovery-db
```
Restore the database to a point before corruption:
```
Recovery Target:
12:44:59
```
Using pgBackRest:
```bash
pgbackrest \
--stanza=payment \
--type=time \
--target="2026-09-14 12:44:59" \
restore
```
# 8. Compare Production and Recovery Database
Two databases are compared:
Production:
```
payment_prod
```
Recovery:
```
payment_restore
```
Example:
Find deleted transactions:
```sql
SELECT *
FROM payment_restore.transactions r
WHERE NOT EXISTS
(
SELECT 1
FROM payment_prod.transactions p
WHERE p.id=r.id
);
```
Find incorrect updates:
```sql
SELECT
p.id,
p.amount,
r.amount
FROM payment_prod.transactions p
JOIN payment_restore.transactions r
ON p.id=r.id
WHERE p.amount <> r.amount;
```
# 9. Selective Data Repair
Only corrupted records are restored.
Example:
Restore deleted transactions:
```sql
INSERT INTO transactions
SELECT *
FROM payment_restore.transactions
WHERE id IN
(
10001,
10002,
10003
);
```
Restore incorrect updates:
```sql
UPDATE transactions t
SET
amount=r.amount,
status=r.status
FROM payment_restore.transactions r
WHERE t.id=r.id;
```
Important:
Do not restore the entire database because valid transactions after the incident time must be preserved.
# 10. Rebuild PostgreSQL Replica
Because the standby database contains corrupted WAL changes, rebuild the replica.
Stop standby:
```bash
systemctl stop postgresql
```
Clean data directory:
```bash
rm -rf $PGDATA/*
```
Clone from Master:
```bash
pg_basebackup \
-h postgres-master \
-D $PGDATA \
-U replicator \
-R
```
Start PostgreSQL:
```bash
systemctl start postgresql
```
# 11. Validation
## Check Master
```sql
SELECT count(*)
FROM transactions;
```
## Check Replica
```sql
SELECT pg_is_in_recovery();
```
Expected:
```
true
```
## Check Replication
Master:
```sql
SELECT
client_addr,
state
FROM pg_stat_replication;
```
Expected:
```
state = streaming
```
# 12. Recovery Completion Checklist
| Item                             | Status |
| -------------------------------- | ------ |
| Application traffic restored     | ☐      |
| Corrupted records repaired       | ☐      |
| Transaction consistency verified | ☐      |
| Replication healthy              | ☐      |
| Monitoring enabled               | ☐      |
| Incident documented              | ☐      |
# 13. Lessons Learned
Preventive improvements:
* Database change approval workflow
* Migration testing before production
* Query review process
* Audit logging
* Backup restore testing
* Monitoring abnormal DELETE/UPDATE operations
* Regular Disaster Recovery exercises
