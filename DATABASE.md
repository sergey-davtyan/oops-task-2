# Innovate Inc. Database Architecture (AWS PostgreSQL)

This README recommends the managed PostgreSQL service on AWS, tailored to Innovate Inc.’s application (Python/Flask backend, React SPA, sensitive user data), and outlines backups, high availability, and disaster recovery.

## Summary Recommendation

- Start with Amazon RDS for PostgreSQL (Multi-AZ) for cost-effective, managed Postgres with strong HA and PITR.
- Add Amazon RDS Proxy between EKS and RDS to handle connection pooling and enable IAM authentication with short‑lived credentials.
- Use customer-managed KMS keys (CMKs) for encryption at rest, enforce TLS in transit, and store credentials in AWS Secrets Manager with rotation.
- Plan an upgrade path to Amazon Aurora PostgreSQL-Compatible if you need higher throughput, faster failover, or multi‑region DR (Aurora Global Database).

Why this path:
- Early-stage friendly: minimal ops, lower cost than Aurora, easy to scale vertically and with read replicas.
- Secure-by-default: private subnets, encryption, IAM auth, and Secrets Manager rotation.
- Clear growth runway: RDS → Aurora migration via Snapshot restore or DMS/Logical replication without app rewrites.

---

## Service Choice and Justification

Option A: Amazon RDS for PostgreSQL (recommended to start)
- Pros: Lower cost, simple operations, Multi‑AZ HA, automated backups/PITR, read replicas (same or cross‑region), supports RDS Proxy, Blue/Green Deployments for near-zero-downtime upgrades.
- Cons: Failover time typically a few minutes; read scaling limited compared to Aurora; global scale requires manual cross‑region replicas.

Option B: Amazon Aurora PostgreSQL-Compatible (consider as you scale)
- Pros: Faster failover (seconds), better read scaling (many readers), storage autoscaling, Global Database for low RPO/RTO multi‑region, I/O-Optimized mode for read-heavy workloads.
- Cons: Higher baseline cost; operational patterns differ (cluster endpoints).

Decision guidance:
- Choose RDS now. Revisit Aurora when: p95 latency or CPU saturates despite vertical scaling; you need sub-minute failover; or multi-region RPO/RTO targets are stringent (<1–2 minutes RTO, near-zero RPO).

---

## Initial Sizing and Configuration (RDS for PostgreSQL)

- Engine: PostgreSQL 15.x (LTS cadence) unless specific extensions dictate otherwise.
- Instance class: Graviton-based for price/performance, e.g., db.m7g.large (prod), db.t4g.medium (dev/staging).
- Storage: gp3 with baseline 1000–3000 IOPS, 100–200 GiB to start; enable storage autoscaling (up to a safe cap).
- Multi-AZ: Yes (DB instance with synchronous standby).
- Deletion protection: Enabled.
- Parameter groups:
  - Force SSL: rds.force_ssl=1
  - Logging: log_min_duration_statement (e.g., 500ms), log_connections, log_disconnections
  - Extensions: pg_trgm, uuid-ossp, pgcrypto, pgaudit (as needed)
- Monitoring: Enhanced Monitoring (Granularity: 1–5s), Performance Insights enabled with 7–31 days retention.

Connection management:
- RDS Proxy with IAM auth and Secrets Manager integration.
- Min/max connections tuned; use transaction pooling if high connection churn (or add PgBouncer if needed).

Networking:
- Private DB subnets across 3 AZs; no public access.
- Security group: allow 5432 only from app security group (EKS).
- TLS required; distribute RDS CA to applications.

---

## Security Controls

- At rest encryption: KMS CMK owned in Security account; strict key policies and grants to the RDS service role and DB account.
- In transit: Enforce TLS; verify server certs in client (psycopg2 sslmode=verify-full).
- Authentication and secrets:
  - Preferred: IAM database authentication via RDS Proxy for application connectivity (short-lived tokens).
  - Alternative: Secrets Manager credentials with automatic rotation (built-in RDS rotation). Mount via Secrets Store CSI in EKS.
- Network isolation: Private subnets, no public IP; security groups as primary control; no 0.0.0.0/0.
- Auditing: pgaudit extension to capture DDL/DCL/selected DML; ship logs to CloudWatch Logs (subscription to SIEM).
- Access model: Break-glass admin user stored in Secrets Manager; day-to-day via IAM auth with least-privilege DB roles.
- Compliance hygiene: S3 buckets for logical exports use Object Lock (compliance mode if required), Block Public Access enabled everywhere.

---

## Backups and Point-in-Time Recovery (PITR)

Automated backups:
- Retention: 14 days to start (adjust per compliance).
- PITR: Enabled by default; allows restore to any second within retention window.
- Backup window: Off-peak hours aligned to region; maintenance window distinct.

AWS Backup (policy-based):
- Centralize backup schedules and lifecycle in AWS Backup with backup vaults protected by KMS CMKs.
- Daily automated backups, weekly/monthly retention tiers (e.g., daily 14 days, weekly 8 weeks, monthly 12 months).
- Cross-account copy: Copy backups to a DR or Log Archive account to protect against account compromise.

Cross-region snapshot copies:
- Nightly encrypted snapshot copy to a secondary region (e.g., us-west-2) with CMK in that region.
- Lifecycle to Glacier for cost control on older backups.

Logical backups (defense in depth):
- Weekly pg_dump/pg_dumpall to S3 in the DR account via a CI/CD job or AWS Batch.
- S3 bucket: Versioned, Object Lock (WORM), SSE-KMS, VPC Gateway Endpoint only (block public).

Backup testing:
- Monthly restore test into a nonprod account from latest snapshot and a random PITR within the last week.
- Automated verification: run schema checksum and basic read/write integration tests.

---

## High Availability (HA)

RDS Multi-AZ:
- Primary and synchronous standby in different AZs; automatic failover during AZ outage or maintenance.
- RTO: typically a few minutes; app must use retry logic and DNS/TCP reconnect.

RDS Proxy:
- Keeps warm connections, reduces failover impact on the application by pooling and replaying (where possible).
- Place Proxy in same subnets as DB; SG allows from app SG; Proxy to DB SG on 5432.

Read scaling:
- Create 1–3 in-region read replicas for heavy read workloads or reporting; update app to route read-only queries when needed.
- Monitor replica lag; set CloudWatch alarms.

Zero/low-downtime changes:
- Use RDS Blue/Green Deployments for major version upgrades or parameter changes with minimal downtime.

---

## Disaster Recovery (DR)

Baseline DR (RDS):
- Cross-region read replica in secondary region for warm standby; asynchronous replication.
- DR drill: quarterly promote replica to primary in secondary region; update Route 53 and application secrets; evaluate RPO (seconds–minutes) and RTO (minutes–tens of minutes).

Backups DR:
- Cross-region, cross-account backup copies allow restore even if both primary DB and account are impacted.
- Runbooks to restore from latest snapshot or PITR in secondary region; re-point application.

Advanced DR (Aurora path):
- Aurora Global Database for near-zero RPO and sub-minute RTO between regions if business requirements tighten.
- Reader endpoint in secondary region for low-latency reads if needed.

Runbook highlights:
- Incident declare → Freeze writes (if possible) → Promote cross-region replica → Update secrets and endpoints → Scale app → Re-enable writes → Backfill any missed messages/events.
- Post-incident: plan failback using DMS/logical replication or snapshot restore.

---

## Maintenance and Upgrades

- Minor version updates: Use maintenance windows; test in staging first; Blue/Green to minimize downtime.
- Major version upgrades: Use Blue/Green Deployments or logical replication to a new instance; verify app compatibility; cut over during low-traffic window.
- Parameter changes: Stage in parameter groups; apply during approved windows; monitor metrics.

Schema migrations:
- Use Alembic/Flask-Migrate in CI/CD.
- Pre-deploy Job in EKS runs migrations with safe, idempotent scripts.
- Back up (snapshot) before schema changes; use feature flags to avoid long locks.

---

## Monitoring and Alerting

Enable and alert on:
- Availability: Failovers, Replica Lag, RDS Proxy health
- Performance: CPU, FreeableMemory, DiskQueueDepth, Read/Write Latency, MaxUsedTransactionIDs, bloat indicators (auto-vacuum logs)
- Capacity: FreeStorageSpace, IOPS throughput, connections vs max_connections
- Errors: Postgres errors in CloudWatch Logs, pgaudit events
- Backups: Snapshot success/failure, AWS Backup job status
- Security: KMS key usage, failed login attempts, parameter drifts

Use:
- CloudWatch Alarms to PagerDuty/Slack
- Performance Insights for SQL tuning
- Enhanced Monitoring for OS metrics

---

## Cost Considerations

- Start small: db.m7g.large Multi-AZ with gp3; scale vertically as needed.
- Use RDS Proxy to pool EKS connections and avoid oversizing DB for connection count.
- Prefer gp3 with tuned IOPS over io1/io2 until proven necessary.
- Turn on storage autoscaling with sensible max to prevent runaway cost.
- Right-size retention: keep hot backups short, archive to Glacier for long-term.
- Pause read replicas when not needed in nonprod.

---

## Implementation Checklist

- Provision RDS PostgreSQL Multi-AZ with:
  - KMS CMK (CMEK)
  - Private subnets and SG rules (app SG → db SG:5432)
  - Parameter group with rds.force_ssl=1 and logging
  - Option group with pgaudit if used
- Create RDS Proxy with IAM auth; attach to DB; grant app IAM role access
- Secrets Manager: create DB admin secret; enable rotation
- AWS Backup: define backup plan, vault, cross-account/region copies
- Cross-region replica or nightly cross-region snapshots (prod)
- CloudWatch: enable log exports, Performance Insights, alarms
- CI/CD: migration job, smoke tests, rollback steps

---

## Future Path (when scale demands)

- Migrate to Aurora PostgreSQL-Compatible:
  - Snapshot restore into Aurora or logical replication for minimal downtime.
  - Adopt Aurora Global Database for multi-region RPO~0 and RTO<1 minute.
  - Use reader endpoints for read scaling; continue using IAM auth and Secrets Manager.
- Consider partitioning, advanced indexing, and query optimization as dataset grows.
- Evaluate data warehousing offloads (e.g., Redshift) for analytics to keep OLTP lean.

This database plan provides secure, managed PostgreSQL with robust backups and HA today, and a pragmatic, low-friction path to global-scale resiliency as Innovate Inc.’s traffic grows.