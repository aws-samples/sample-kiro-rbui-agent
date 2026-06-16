# RBUI Resilience Assessment Report — Example Output
**Account:** 123456789012 | **Region:** us-east-1 | **Date:** 2026-06-16

---

## Overall Score: 42/100 (FAIR)

| Dimension | Score | Max |
|-----------|-------|-----|
| Regional HA | 14 | 25 |
| Data Protection | 13 | 25 |
| Cross-Region DR | 5 | 25 |
| Application Resilience | 10 | 25 |

---

## Infrastructure Inventory

| Resource | Type | Engine | Version | Size | Encrypted | Multi-AZ | Readers | Del Protection |
|----------|------|--------|---------|------|-----------|----------|---------|---------------|
| prod-db-1 | RDS Instance | PostgreSQL | 16.4 | 500 GB gp3 | ✅ CMK | ✅ Yes | 0 | ✅ On |
| analytics-cluster | Aurora Cluster | aurora-postgresql | 16.4 | db.r6g.xlarge | ✅ CMK | N/A | 1 writer only | ❌ Off |
| staging-db | RDS Instance | PostgreSQL | 15.8 | 50 GB gp2 | ❌ No | ❌ No | 0 | ❌ Off |

---

## Blockers Detected

| Severity | Blocker ID | Resource | Description | RTO/RPO Impact |
|----------|-----------|----------|-------------|---------------|
| 🔴 CRITICAL | FT-07 | analytics-cluster | Aurora single writer, no readers — no automatic failover target | RTO: 10-15 min |
| 🔴 CRITICAL | FT-09 | staging-db | Single-AZ — AZ failure = full outage requiring snapshot restore | RTO: 15-30 min |
| 🟠 HIGH | EN-01, EN-04 | staging-db | Unencrypted — blocks all cross-region DR paths | Blocks DR |
| 🟠 HIGH | CR-01 | ALL resources | No cross-region DR configured — regional failure = total outage | RTO: hours |
| 🟠 HIGH | — | analytics-cluster | Deletion protection OFF — accidental deletion risk | RPO: up to 24h |
| 🟡 MEDIUM | CR-05 | ALL | Mixed versions (15.8 vs 16.4) — blocks Global Database | Must align versions |
| 🟡 MEDIUM | AL-01 | ALL | No RDS Proxy detected | Failover depends on DNS propagation |
| 🟢 LOW | SR-05 | staging-db | Using default parameter group | Performance not optimized |

---

## Realistic RTO/RPO Analysis (Current State)

| Resource | Recovery Scenario | Actual RPO | Actual RTO | Target | Gap |
|----------|------------------|-----------|-----------|--------|-----|
| **prod-db-1** (500GB) | AZ failure | 0 (sync) | **2-3 min** | <1 min | ⚠️ 1-2 min |
| **prod-db-1** (500GB) | Regional failure | **up to 24h** | **2-4 hours** | <15 min | ❌ critical |
| **analytics-cluster** | Writer crash | 5 min (PITR) | **10-15 min** | <30 sec | ❌ 14.5 min |
| **staging-db** (50GB) | AZ failure | 5 min (PITR) | **15-30 min** | <5 min | ❌ 25 min |

### RTO Breakdown: prod-db-1 (AZ Failure with Multi-AZ)

```
  - Failover detection + switch:    ~60-120 seconds
  - DNS propagation (60s TTL):      ~60 seconds
  - Connection pool refresh:        ~5-10 seconds
  ─────────────────────────────────────────────
  TOTAL RTO:                        ~2-3 minutes
```

### RTO Breakdown: analytics-cluster (Writer Crash, No Reader)

```
  - Aurora detects writer failure:  ~10-30 seconds
  - No reader to promote:           must provision new instance
  - New instance launch:            ~5-10 minutes
  - Storage volume reattachment:    ~1-2 minutes
  - Connection re-establishment:    ~30 seconds
  ─────────────────────────────────────────────
  TOTAL RTO:                        ~10-15 minutes
```

---

## Prioritized Remediation Plan

### P1 — IMMEDIATE: In-Region HA (Zero Downtime)

```bash
# Add Aurora reader to analytics-cluster
aws rds create-db-instance \
  --db-instance-identifier analytics-cluster-reader-1 \
  --db-instance-class db.r6g.xlarge \
  --engine aurora-postgresql \
  --db-cluster-identifier analytics-cluster \
  --availability-zone us-east-1b \
  --region us-east-1

# Enable deletion protection
aws rds modify-db-cluster \
  --db-cluster-identifier analytics-cluster \
  --deletion-protection \
  --apply-immediately \
  --region us-east-1

# Enable Multi-AZ for staging-db
aws rds modify-db-instance \
  --db-instance-identifier staging-db \
  --multi-az \
  --apply-immediately \
  --region us-east-1
```

### P2 — Data Protection (This Week)

```bash
# Encrypt staging-db (requires maintenance window)
# Step 1: Snapshot
aws rds create-db-snapshot \
  --db-instance-identifier staging-db \
  --db-snapshot-identifier staging-db-pre-encrypt

# Step 2: Copy with encryption
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier staging-db-pre-encrypt \
  --target-db-snapshot-identifier staging-db-encrypted \
  --kms-key-id alias/rds-cmk

# Step 3: Restore encrypted instance
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier staging-db-encrypted \
  --db-snapshot-identifier staging-db-encrypted \
  --db-instance-class db.t3.medium
```

### P3 — Cross-Region DR (30 Days)

```bash
# Enable cross-region automated backup for prod-db-1
aws rds start-db-instance-automated-backups-replication \
  --source-db-instance-arn arn:aws:rds:us-east-1:123456789012:db:prod-db-1 \
  --kms-key-id alias/rds-cmk-us-west-2 \
  --region us-west-2

# Setup Aurora Global Database for analytics-cluster
aws rds create-global-cluster \
  --global-cluster-identifier analytics-global \
  --source-db-cluster-identifier arn:aws:rds:us-east-1:123456789012:cluster:analytics-cluster \
  --region us-east-1
```

---

## Cost Impact Summary

| Remediation | Monthly Cost Δ |
|-------------|----------------|
| Aurora reader for analytics-cluster (db.r6g.xlarge) | +$380/mo |
| Multi-AZ for staging-db (db.t3.medium) | +$35/mo |
| Cross-region backup (prod-db-1, 500GB) | +$50/mo |
| Aurora Global Database secondary | +$500/mo |
| **Net P1** | **+$415/mo** |
| **Net All** | **+$965/mo** |

---

## Score Progression

| Stage | Score | Rating | Worst-Case RTO | RPO |
|-------|-------|--------|---------------|-----|
| Current | 42/100 | FAIR | Hours (no DR) | Up to 24h |
| After P1 | 62/100 | GOOD | 2-3 min | 5 min |
| After P1+P2 | 72/100 | GOOD | 2-3 min | 5 min |
| After All | 88/100 | EXCELLENT | <1 min | <1 sec |
