# Sandbox Migration Shield — Architecture Summary

**Product:** sandbox-migration-shield  
**Repo:** `vladarchitectservicenow-oss/sandbox-migration-shield`  
**Scope:** `x_sandbox_migration_shield`  
**Release Target:** Australia (May 2026)  
**Author:** Vladimir Kapustin  
**Date:** 2026-06-01

## 1. Product Purpose

Sandbox Migration Shield prevents data corruption and configuration drift when refreshing ServiceNow sub-production instances (sandbox, dev, test) from production. It captures a pre-refresh snapshot of the sandbox environment — customizations, active update sets, in-progress developments, data exclusions, and instance-specific configurations — and compares it against the post-refresh state. The shield then generates a reconciliation report identifying what was **lost, changed, or preserved** during the clone/migration, enabling teams to restore critical customizations that would otherwise be silently overwritten.

## 2. Architecture Layers

### Layer 1 — Pre-Refresh Snapshot Engine
- Runs on sandbox instance before clone
- Captures: all update sets (in-progress + complete), system properties overridden from global defaults, custom tables/fields/business rules, scripted REST APIs, scheduled jobs, email configurations
- Stores snapshot as encrypted JSON in `x_sandbox_migration_shield_snapshot` table
- Exports snapshot to external storage (attachment or REST endpoint) as backup in case sandbox is fully wiped

### Layer 2 — Post-Refresh Delta Engine
- Runs on sandbox instance after clone
- Re-scans the same configuration categories
- Compares against pre-refresh snapshot stored externally (or re-imported)
- Computes delta: ADDED (new from prod), DELETED (was in sandbox, lost during clone), MODIFIED (changed), PRESERVED (unchanged)

### Layer 3 — Reconciliation & Restore Engine
- Generates categorized reconciliation report
- For DELETED items: provides one-click restore via Update Set XML re-import
- For MODIFIED items: shows diff view (before/after)
- For ADDED items: flags production artifacts that leaked into sandbox (optional cleanup)
- Priority-based restore: update sets first, then system properties, then custom code

### Layer 4 — Governance & Audit
- All snapshots and reconciliations logged with timestamps, user, and scope
- Audit trail for compliance (SOX, GDPR) showing what was restored after each clone
- Dashboard showing sandbox refresh health over time (drift trend)

## 3. Data Model

| Table | Purpose |
|-------|---------|
| `x_sandbox_migration_shield_snapshot` | Pre/post-refresh configuration snapshots (encrypted JSON) |
| `x_sandbox_migration_shield_delta` | Per-item delta records (ADDED/DELETED/MODIFIED/PRESERVED) |
| `x_sandbox_migration_shield_config` | Shield configuration (exclusion lists, priority rules, external storage endpoint) |
| `x_sandbox_migration_shield_audit` | Audit log for all snapshot/restore operations |
| `x_sandbox_migration_shield_update_set_xml` | Stored XML for deleted update sets (for restore) |

## 4. Security Model

- Snapshots encrypted at rest (GlideEncrypter) — sandbox configurations may contain sensitive connection details
- Read-only on production — shield never writes to production instance
- External snapshot export uses HTTPS with API key authentication
- Role-based access: `x_sandbox_migration_shield.admin` (configure, restore), `.viewer` (read reports)
- GDPR-aware: snapshot data tagged with retention policy (auto-purge after 90 days)

## 5. Component Diagram

```
┌───────────────────────────────────────────────────────────┐
│                Sandbox Migration Shield                    │
│                                                           │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────────┐ │
│  │ Pre-Refresh  │   │ Post-Refresh │   │ Reconciliation│ │
│  │ Snapshot     │──▶│ Delta Engine │──▶│ & Restore     │ │
│  │ Engine       │   │              │   │ Engine        │ │
│  └──────────────┘   └──────────────┘   └───────────────┘ │
│         │                  │                    │          │
│         ▼                  ▼                    ▼          │
│  ┌───────────────────────────────────────────────────┐   │
│  │  Snapshot Storage (encrypted JSON + attachments)   │   │
│  └───────────────────────────────────────────────────┘   │
│                                                           │
│  External Backup (REST endpoint / S3)                     │
└───────────────────────────────────────────────────────────┘
```

## 6. Key Design Decisions

1. **External snapshot backup** — if sandbox is fully wiped during clone (some orgs do this), the in-instance snapshot table is destroyed. External backup via REST ensures recovery.
2. **Delta-first design** — don't restore everything blindly. Show what changed, let admin choose what to restore.
3. **Update set priority** — in-progress development is the highest-value artifact. Restore update sets first, then system properties, then custom code.
4. **Production-read-only** — shield operates entirely within sandbox. Never connects to production.
5. **Encryption at rest** — snapshot data may contain instance-specific credentials and configuration details that should not be inspectable.
