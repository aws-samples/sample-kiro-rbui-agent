# RBUI Blocker Catalog Reference

## Overview

The RBUI agent checks for **57 hidden resilience blockers** across **7 categories** that create gaps between documented AWS DR capabilities and actual achievable RTO/RPO.

---

## Category 1: Failover Timing (FT) — 9 Blockers

These affect in-region high availability and determine how long failover actually takes.

| ID | Blocker | Severity | RTO Impact |
|----|---------|----------|------------|
| FT-01 | RDS Multi-AZ failover takes 60-120s (single standby) | INFO | 1-2 min minimum downtime |
| FT-02 | Multi-AZ with two readable standbys: <35s (PostgreSQL/MySQL only) | INFO | Engine-limited availability |
| FT-03 | Large transactions increase failover time beyond 120s | MEDIUM | Unpredictable failover under load |
| FT-04 | Aurora DNS TTL = 5s, but client caching extends staleness | MEDIUM | Stale routing window |
| FT-05 | RDS (non-Aurora) DNS CNAME TTL = 60s | MEDIUM | 60s stale routing after failover |
| FT-06 | JVM default DNS TTL can be INFINITE | HIGH | Java apps never reconnect |
| FT-07 | Aurora single-writer without readers: no failover target | CRITICAL | RTO: 10-15 min |
| FT-08 | Aurora secondary cluster readers restart during primary failover | HIGH | Secondary unavailable during events |
| FT-09 | Single-AZ RDS: AZ failure = snapshot restore required | CRITICAL | RTO: 30-60+ min |

---

## Category 2: Snapshot Restore Constraints (SR) — 10 Blockers

These affect recovery time when restoring from backups.

| ID | Blocker | Severity | Impact |
|----|---------|----------|--------|
| SR-01 | Snapshot restore uses lazy loading from S3 | MEDIUM | "Available" ≠ performant |
| SR-02 | Changing storage type during restore slows the process | LOW | Adds restore time |
| SR-03 | Cannot restore to existing instance — always creates new | HIGH | Endpoint changes required |
| SR-04 | Cannot reduce allocated storage on restore | LOW | Storage locked at snapshot time |
| SR-05 | Default parameter group assigned on restore | MEDIUM | Custom tuning lost |
| SR-06 | Default security group assigned on restore | HIGH | DB may be unreachable |
| SR-07 | Aurora PITR restores cluster only — instances added separately | MEDIUM | +5-10 min per instance |
| SR-08 | PITR granularity: transaction logs uploaded every 5 min | MEDIUM | 5-min RPO gap minimum |
| SR-09 | Cannot restore from shared AND encrypted snapshot directly | MEDIUM | Must copy first |
| SR-10 | Restore time scales with database size | INFO | 400GB ≈ 15-30 min to available |

---

## Category 3: Encryption Constraints (EN) — 6 Blockers

These create cascading failures that block entire DR paths.

| ID | Blocker | Severity | Impact |
|----|---------|----------|--------|
| EN-01 | Cannot enable encryption on existing unencrypted instance | HIGH | Requires snapshot→encrypt→restore migration |
| EN-02 | KMS key cannot be changed directly | MEDIUM | Key rotation requires full migration |
| EN-03 | Cross-region snapshot copy requires re-encryption | MEDIUM | Adds time + pre-provisioned key needed |
| EN-04 | Unencrypted DBs cannot use cross-region replicas/Global DB | HIGH | Blocks ALL cross-region DR |
| EN-05 | AWS-managed key (aws/rds) blocks cross-account backup copy | MEDIUM | Must use customer-managed CMK |
| EN-06 | KMS inaccessible-encryption-credentials is TERMINAL for Global DB | CRITICAL | No recovery if key access lost |

---

## Category 4: KMS API Throttling (KT) — 4 Blockers

These create bottlenecks during mass DR events.

| ID | Blocker | Severity | Impact |
|----|---------|----------|--------|
| KT-01 | Symmetric operations quota: 5,500 req/s (most regions) | MEDIUM | Parallel restores share this |
| KT-02 | KMS quota shared across ALL services using same key | HIGH | RDS competes with S3, EBS, Lambda |
| KT-03 | Exceeding quota returns ThrottlingException | HIGH | Restores stall or fail |
| KT-04 | CreateGrant quota: 50 req/s — each RDS operation needs one | HIGH | Bottleneck for fleet restores |

---

## Category 5: Cross-Region DR Constraints (CR) — 14 Blockers

These affect cross-region disaster recovery capabilities.

| ID | Blocker | Severity | Impact |
|----|---------|----------|--------|
| CR-01 | Aurora: no cross-region automated backup replication | HIGH | Must use Global DB or manual copies |
| CR-02 | Multi-AZ DB clusters: no cross-region automated backups | HIGH | Architecture gap |
| CR-03 | Max 20 cross-region automated backup replications | MEDIUM | Large fleet must prioritize |
| CR-04 | Not all source→destination region pairs supported | MEDIUM | DR region choice constrained |
| CR-05 | Global DB switchover requires same major+minor version | CRITICAL | Version mismatch blocks DR |
| CR-06 | Some engines require identical patch levels | HIGH | Patch drift breaks DR silently |
| CR-07 | Global DB does NOT support Backtrack | LOW | No fast rollback with global |
| CR-08 | Global DB: no Auto Scaling for secondary clusters | MEDIUM | Secondary may be under-provisioned |
| CR-09 | Cannot apply custom PG during major version upgrade | MEDIUM | Manual PG application per region |
| CR-10 | Secrets Manager NOT supported with Global DB | MEDIUM | Credential management gap |
| CR-11 | Auto minor version upgrade has NO EFFECT on global clusters | MEDIUM | Manual coordination required |
| CR-12 | Primary from RDS PG replica cannot create secondary | LOW | Specific migration path blocked |
| CR-13 | Cannot stop/start individual clusters in Global DB | LOW | Cost management limited |
| CR-14 | Replication is async — sub-second typical but NOT guaranteed | MEDIUM | Lag can exceed 1s under load |

---

## Category 6: Application-Layer Resilience Gaps (AL) — 6 Blockers

These affect how applications handle database failover events.

| ID | Blocker | Severity | Impact |
|----|---------|----------|--------|
| AL-01 | Without RDS Proxy/AWS JDBC Driver, failover = DNS dependent | HIGH | 5-60s stale routing |
| AL-02 | Connection pools hold stale connections after failover | HIGH | Errors until pool cycles |
| AL-03 | TCP keepalive defaults (2+ hours) miss dead connections | MEDIUM | Minutes to detect |
| AL-04 | RDS Proxy on secondary fails after global failover | MEDIUM | Must redirect to new primary |
| AL-05 | Write forwarding adds latency on secondary writes | LOW | Not a local-write replacement |
| AL-06 | Cluster cache management not supported for PG secondary | MEDIUM | Cold buffer pool after failover |

---

## Category 7: Account-Level Service Quotas (QT) — 20 Blockers

These silently block DR operations when limits are reached.

| ID | Blocker | Default Limit | Severity |
|----|---------|---------------|----------|
| QT-01 | Manual DB cluster snapshots | 100 | HIGH |
| QT-02 | Manual DB instance snapshots | 100 | HIGH |
| QT-03 | DB instances per region | 40 | CRITICAL |
| QT-04 | DB clusters per region | 40 | CRITICAL |
| QT-05 | Total storage (all instances) | 100 TB | MEDIUM |
| QT-06 | Cross-region backup replications | 20 | MEDIUM |
| QT-07 | Concurrent cross-region snapshot copies | 5 | HIGH |
| QT-08 | DB parameter groups | 50 | MEDIUM |
| QT-09 | DB subnet groups | 50 | MEDIUM |
| QT-10 | Aurora Global Databases | 5 | MEDIUM |
| QT-11 | Read replicas per source | 5 (RDS) / 15 (Aurora) | LOW |
| QT-12 | VPC security groups per instance | 5 | LOW |
| QT-13 | Event subscriptions | 20 | LOW |
| QT-14 | Reserved DB instances | 40 | LOW |
| QT-15 | KMS CreateGrant API calls | 50 req/sec | HIGH |
| QT-16 | KMS grants per key | 50,000 | LOW |
| QT-17 | Option groups | 20 | MEDIUM |
| QT-18 | Custom endpoints per cluster | 5 | LOW |
| QT-19 | Proxies per account | 20 | MEDIUM |
| QT-20 | IAM roles (monitoring/proxy) | 1,000 | LOW |

---

## Detection Priority

When assessing an account, apply rules in this order:

1. **CRITICAL** — immediate operational risk (FT-07, FT-09, CR-05, QT-03, QT-04, EN-06)
2. **HIGH** — blocks DR paths or causes extended outages (EN-01, EN-04, CR-01, AL-01, AL-02)
3. **MEDIUM** — degrades recovery or limits options (SR-05, SR-08, KT-01, CR-14)
4. **LOW** — suboptimal but not blocking (SR-02, CR-07, CR-13)
