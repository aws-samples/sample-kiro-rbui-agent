# RBUI Scoring Methodology

## Overview

Each RDS/Aurora resource is scored 0-100 across four dimensions (25 points each). The overall account score is the weighted average across all active resources.

---

## Score Ranges

| Score | Rating | Meaning |
|-------|--------|---------|
| 80-100 | EXCELLENT | Multi-region, encrypted, auto-failover, tested DR |
| 60-79 | GOOD | Regional HA present, some DR gaps, mostly encrypted |
| 40-59 | FAIR | Basic HA (Multi-AZ) but no cross-region, some gaps |
| 20-39 | POOR | Single-AZ, minimal backup, major gaps |
| 0-19 | CRITICAL | No HA, no DR, unencrypted, at risk of total loss |

---

## Dimension 1: Regional HA (25 points)

| Criterion | Points | Detection |
|-----------|--------|-----------|
| Multi-AZ enabled | +10 | `multiAZ == true` or Aurora cluster with instances in multiple AZs |
| Aurora with 2+ readers (or RDS 2-standby) | +8 | `clusterMembers.count >= 3` or Multi-AZ with two standbys |
| Deletion protection ON | +4 | `deletionProtection == true` |
| Backup retention ≥14 days | +3 | `backupRetentionPeriod >= 14` |

---

## Dimension 2: Data Protection (25 points)

| Criterion | Points | Detection |
|-----------|--------|-----------|
| Encrypted at rest | +10 | `storageEncrypted == true` |
| Customer-managed KMS key | +5 | KMS key ARN does not contain `alias/aws/rds` |
| Cross-region backup replication | +7 | Cross-region automated backup or snapshot copy configured |
| PITR enabled (retention >0) | +3 | `backupRetentionPeriod > 0` |

---

## Dimension 3: Cross-Region DR (25 points)

| Criterion | Points | Detection |
|-----------|--------|-----------|
| Global Database or cross-region replica | +15 | `globalCluster` membership or cross-region read replica |
| Same engine version across regions | +5 | Primary and secondary version match |
| DR tested within last 90 days | +5 | Manual verification (ask user) |

---

## Dimension 4: Application Resilience (25 points)

| Criterion | Points | Detection |
|-----------|--------|-----------|
| RDS Proxy or AWS JDBC Driver | +10 | Proxy associated with instance/cluster |
| TCP keepalive configured | +5 | Parameter group: `tcp_keepalives_idle <= 60` |
| DNS TTL ≤5s (or proxy bypass) | +5 | Aurora native (5s) or proxy endpoint |
| Failover runbook documented | +5 | Manual verification (ask user) |

---

## RTO/RPO Formulas

### Snapshot-Based Recovery

```
RTO = snapshot_locate_time (1-2 min)
    + restore_initiation_time (1 min)
    + instance_boot_time (size-dependent)
    + parameter_group_reapply (2-3 min)
    + security_group_reapply (1 min)
    + dns_propagation (1 min)
    + application_reconnection (5-10 min)
    + data_warmup_time (optional)

Size-based boot time:
  < 100 GB:    5-10 min
  100-500 GB:  10-20 min
  500 GB-1 TB: 20-40 min
  > 1 TB:      40-90 min

RPO:
  With PITR: 5 minutes maximum
  Snapshot only: up to 24 hours
```

### Multi-AZ Failover

```
RTO = failover_detection (10-30s)
    + dns_update (immediate for Aurora / 60s for RDS)
    + client_dns_cache_expiry (varies)
    + connection_pool_drain (5-10s)

Results:
  RDS single standby: 60-120 seconds
  RDS two standbys: <35 seconds
  Aurora with readers: 15-30 seconds
  Aurora without readers: 10-15 minutes

RPO = 0 (synchronous replication)
```

### Aurora Global Database

```
RTO = failure_detection (10-30s)
    + switchover/failover execution (<60s)
    + dns_propagation (5s × 2-3 cycles)
    + application_reconnection (5-10s)

Results:
  Planned switchover: <1 minute
  Unplanned failover: 1-2 minutes

RPO = replication_lag (typically <1 second)
```

### Quota-Adjusted Mass DR

```
# When concurrent snapshot copy limit (5) affects fleet DR:
adjusted_rto = base_rto + (batch_position / 5) * avg_copy_time

# When instance quota blocks restore:
adjusted_rto = INFINITY (until quota increase approved)
```
