# PostgreSQL Payment Service Operations Guide
## 1. Purpose
This document describes the daily operational procedures for managing the PostgreSQL Payment Service.
The objectives are:
* Maintain database availability
* Detect issues before service impact
* Verify backup reliability
* Monitor replication health
* Ensure production database stability
# 2. Daily DBA Health Check
The DBA team should perform daily health checks on PostgreSQL environments.
## Database Availability
Check database status:
bash
systemctl status postgresql
or in Kubernetes:
bash
kubectl get pods -n payment-db
Expected result:
* PostgreSQL pods are running
* Database service is available
# 3. Database Connection Check
Check active connections:
sql
SELECT
count(*)
FROM pg_stat_activity;
Review:
* Current connections
* Idle sessions
* Long-running queries
Check long-running queries:
sql
SELECT
pid,
usename,
state,
query_start,
query
FROM pg_stat_activity
WHERE state <> 'idle'
ORDER BY query_start;
# 4. Replication Health Check
The PostgreSQL environment uses Streaming Replication.
## Primary Check
Run on Primary:
sql
SELECT
client_addr,
state,
sent_lsn,
write_lsn,
flush_lsn,
replay_lsn
FROM pg_stat_replication;
Expected:
text
state = streaming
## Standby Check
Run on Replica:
sql
SELECT pg_is_in_recovery();
Expected:
text
true
# 5. WAL and Replication Monitoring
Monitor:
* WAL generation rate
* Replication lag
* Replica replay status
Important metrics:
text
replication_lag_seconds
Alert when:
text
Replication lag > acceptable threshold
# 6. Backup Verification
Backup status must be checked daily.
Example:
bash
pgbackrest info
Verify:
* Latest backup exists
* Backup completed successfully
* WAL archive is available
* Storage capacity is sufficient
# 7. PostgreSQL Log Review
Review PostgreSQL logs for:
## Critical Errors
Examples:
* PANIC
* ERROR
* FATAL
* Deadlocks
* Connection failures
Example:
bash
grep -i "error" postgresql.log
# 8. Performance Monitoring
Monitor:
## Database Metrics
* CPU usage
* Memory usage
* Disk usage
* Database size
## PostgreSQL Metrics
* Active connections
* Transaction rate
* Query latency
* Lock waits
Example:
sql
SELECT *
FROM pg_locks
WHERE NOT granted;
# 9. Storage Monitoring
Check disk usage:
bash
df -h
Important:
Monitor:
* PostgreSQL data directory
* WAL directory
* Backup storage
Low disk space can cause:
* Database failures
* WAL write failures
* Recovery issues
# 10. Operational Checklist
| Check                  | Frequency |
| ---------------------- | --------- |
| Database availability  | Daily     |
| Replication status     | Daily     |
| Backup verification    | Daily     |
| PostgreSQL logs review | Daily     |
| Disk space check       | Daily     |
| Performance review     | Weekly    |
| Disaster Recovery test | Regularly |
# 11. Incident Escalation
If a critical issue is detected:
1. Collect database evidence
2. Check application impact
3. Stop harmful operations
4. Follow recovery runbook
5. Document incident
# Conclusion

A consistent operational process helps maintain PostgreSQL reliability and reduces the impact of production incidents.
