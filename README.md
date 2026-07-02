# RBUI Agent — RDS/Aurora Resilience Assessment for Kiro CLI

A read-only Kiro CLI agent that uncovers hidden resilience blockers in your AWS RDS and Aurora infrastructure — the gap between documented DR capabilities and actual achievable RTO/RPO.

## Overview

The **RBUI (Resilience Blockers Underneath Iceberg)** agent performs automated topology-aware assessments of RDS/Aurora databases, identifying constraints that silently prevent meeting stated recovery targets.

> **What is Kiro?** [Kiro](https://kiro.dev) is an AI-powered IDE. Agents are defined in `.kiro/agents/` and provide specialized capabilities. This repo contains a Kiro CLI agent that you invoke with `kiro chat` to assess your database resilience posture.

```
COLLECT → CLASSIFY → CALCULATE → REPORT
```

The agent discovers your RDS topology, maps it against 57 known blockers, computes realistic RTO/RPO values, and produces a prioritized remediation plan with AWS CLI commands that are for manual execution only.
The prerequisite of this agent is to install Kiro CLI from https://kiro.dev/

## Quick Start

1. **Clone this repo** into your project:
   ```bash
   git clone https://github.com/belmacanik-ctrl/sample-kiro-rbui-agent.git
   cd sample-kiro-rbui-agent
   ```

2. **Start Kiro CLI:**
   ```bash
   kiro-cli  chat  --agent rbui-agent
   ```

3. **Invoke the agent** (Ctrl+Shift+B) or ask:
   ```
   Assess RDS resilience for account 123456789012 in us-east-1
   ```

## What It Detects

### 57 Blockers Across 7 Categories

| Category | ID Prefix | Count | Example |
|----------|-----------|-------|---------|
| Failover Timing | FT- | 9 | Aurora single-writer without readers → 10-15 min RTO |
| Snapshot Restore | SR- | 10 | Lazy loading from S3 — "available" ≠ performant |
| Encryption | EN- | 6 | Unencrypted DB blocks ALL cross-region DR paths |
| KMS Throttling | KT- | 4 | Parallel encrypted restores share 5,500 req/s quota |
| Cross-Region DR | CR- | 14 | Version mismatch silently breaks Global Database failover |
| Application Layer | AL- | 6 | JVM DNS cache can be infinite — never reconnects |
| Account Quotas | QT- | 20 | Only 5 concurrent cross-region snapshot copies allowed |

### Scoring (0-100, Four Dimensions × 25 pts)

| Dimension | What It Measures |
|-----------|-----------------|
| Regional HA | Multi-AZ, reader count, deletion protection, backup retention |
| Data Protection | Encryption, CMK, cross-region backups, PITR |
| Cross-Region DR | Global Database, version alignment, tested DR |
| Application Resilience | RDS Proxy, TCP keepalive, DNS TTL, runbooks |

### RTO Calculation

| Scenario | RTO Range |
|----------|-----------|
| Aurora with readers | 15-30 seconds |
| Multi-AZ failover (single standby) | 60-120 seconds |
| Multi-AZ failover (two standbys) | <35 seconds |
| Aurora without readers | 10-15 minutes |
| Snapshot restore (<100GB) | 15-30 minutes |
| Snapshot restore (100-500GB) | 30-60 minutes |
| Snapshot restore (>500GB) | 60-180 minutes |
| Aurora Global Database (planned) | <1 minute |
| Aurora Global Database (unplanned) | 1-2 minutes |

## Project Structure

```
sample-kiro-rbui-agent/
├── .kiro/
│   └── agents/
│       ├── rbui-agent.json       # Agent definition (name, tools, prompt, shortcut)
│       └── skills.md             # 57 blockers, detection rules, RTO formulas, scoring
├── docs/
│   ├── BLOCKER-CATALOG.md        # Full blocker reference
│   └── SCORING.md                # Scoring methodology and formulas
├── examples/
│   ├── sample-topology.json      # Example RDS topology input
│   └── sample-assessment-output.md  # Example assessment report
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

## Required IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters",
        "rds:DescribeGlobalClusters",
        "rds:DescribeDBClusterSnapshots",
        "rds:DescribeDBSnapshots",
        "rds:DescribeAccountAttributes",
        "kms:DescribeKey",
        "kms:ListAliases",
        "service-quotas:GetServiceQuota",
        "service-quotas:ListServiceQuotas"
      ],
      "Resource": "*"
    }
  ]
}
```

## How to Customize

- **Add blockers:** Edit `.kiro/agents/skills.md` — add entries to the category tables and detection rules
- **Adjust scoring:** Modify the scoring dimensions under `ASSESSMENT SCORING MATRIX`
- **Change remediation:** Update `REMEDIATION PLAYBOOK TEMPLATES` with your preferred commands

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This project is licensed under the MIT-0 License. See the [LICENSE](LICENSE) file.
