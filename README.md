# PostgreSQL Service Disaster Recovery
## Overview
This project demonstrates a real-world PostgreSQL disaster recovery scenario for a payment service running on Kubernetes.
The objective is to simulate a production database incident where incorrect SQL operations corrupt transactional data and implement a safe recovery strategy with minimum data loss.
# Scenario
A PostgreSQL production database is running with:
* PostgreSQL Primary-Standby Streaming Replication
* Kubernetes Deployment
* 24/7 Payment Transaction Processing
At 12:45 PM,an incorrect migration script caused:
* Wrong DELETE operations
* Incorrect UPDATE operations
* Duplicate or invalid transaction records
Because streaming replication was enabled,corrupted WAL changes were replicated to the standby server.
A standby replica cannot be used as a backup source because logical corruption can be replicated together with WAL changes.
# Goals
The recovery process must achieve:
* Minimum Data Loss (RPO close to zero)
* Minimum Downtime (Low RTO)
* Selective Data Recovery
* Preserve valid transactions after incident time
* Restore healthy replication
# Architecture
                  Payment Application
                         |
                         |
              PostgreSQL Primary
                         |
          --
          |                              |
          |                              |
 Streaming Replication             WAL Archive
          |                              |
          |                              |
 PostgreSQL Standby             Backup Repository
                                         |
                                         |
                                PITR Recovery DB
# Technology Stack
| Component   | Technology            |
| -- |  |
| Database    | PostgreSQL 16         |
| Platform    | Kubernetes            |
| Cluster     | Kind                  |
| Replication | Streaming Replication |
| Backup      | pgBackRest            |
| Monitoring  | Prometheus + Grafana  |
# Recovery Strategy
The recovery approach:
1. Stop application writes
2. Freeze database changes
3. Analyze corrupted data
4. Restore database using Point In Time Recovery
5. Compare corrupted records
6. Repair only affected transactions
7. Rebuild replication
8. Validate database health
# Repository Structure
.
├── README.md
├── docs
├── kubernetes
├── postgres
├── backup
├── scripts
├── monitoring
└── evidence
# How to Run
The project workflow:
1. Create Kubernetes cluster
2. Deploy PostgreSQL Primary and Standby
3. Initialize payment database
4. Generate transaction data
5. Simulate corruption scenario
6. Execute recovery procedure
7. Validate database health
# Documentation
Detailed documentation:
* Incident Analysis
* Recovery Runbook
* Architecture Design
* Disaster Recovery Plan
* Preventive Actions
* Operations Guide
* Testing Evidence
# Monitoring
The project includes monitoring examples for:
* Replication lag
* Abnormal DELETE rate
* Abnormal UPDATE rate
* Database availability
* Backup status
# Author
Sarvenaz Sardari
Senior Database Engineer | Database Platform Engineer
