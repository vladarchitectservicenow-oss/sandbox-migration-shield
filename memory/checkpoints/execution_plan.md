# Sandbox Migration Shield — Execution Plan

**Date:** 2026-06-01  
**Author:** Vladimir Kapustin  
**Target Release:** Australia (May 2026)

## Phase Overview

| Phase | Name | Duration | Status |
|-------|------|----------|--------|
| 1 | Documentation & Architecture | 2 days | ✅ COMPLETE (2026-06-01) |
| 2 | Validation Suite | 2 days | ✅ COMPLETE (2026-06-01) |
| 3 | Core Snapshot Engine | 5 days | Planned |
| 4 | Delta & Reconciliation Engine | 4 days | Planned |
| 5 | External Backup Module | 3 days | Planned |
| 6 | Restore Engine | 3 days | Planned |
| 7 | Testing & QA | 4 days | Planned |
| 8 | PDI Smoke Test | 2 days | Planned |
| 9 | Release Packaging | 1 day | Planned |

**Total estimated:** 26 days

## Phase 3 — Core Snapshot Engine

### Task 3.1: Snapshot Collector (Script Include)
- Implement `SandboxSnapshot.collect()` — iterates all tracked tables
- Categories: Update Sets, System Properties, Custom Tables/Fields, Script Includes, REST Endpoints, Scheduled Jobs, Email Configs, UI Policies, UI Actions
- Progress tracking per category: `current: {category, offset, total}`
- Output: structured JSON blob with metadata (timestamp, instance_id, version, checksum)

### Task 3.2: Credential Redaction
- Scan sys_properties for credential-like patterns: `*password*`, `*key*`, `*token*`, `*secret*`, `*auth*`
- Redact values before snapshot storage: replace with `[REDACTED]`
- Redaction audit log entry: property_sys_id, redacted_field

### Task 3.3: Snapshot Storage
- Encrypt JSON blob via GlideEncrypter
- Store in `x_sandbox_migration_shield_snapshot` table
- Also store as attachment (sys_attachment) for export
- Generate SHA-256 checksum

### Task 3.4: External Backup
- POST encrypted snapshot to configured REST endpoint
- Verify 200 OK response
- If REST fails: log warning, attachment remains as sole backup
- Health check: test external endpoint connectivity before snapshot

## Phase 4 — Delta & Reconciliation Engine

### Task 4.1: Post-Clone Scanner
- Re-run snapshot collection on post-clone instance
- Compare against pre-snapshot (loaded from external storage or attachment)
- Classification: ADDED (in post, not in pre), DELETED (in pre, not in post), MODIFIED (changed), PRESERVED (identical)

### Task 4.2: Conflict Detection
- For MODIFIED items: show before/after diff
- For DELETED update sets: check if update set name collides with any post-clone update set
- Flag conflicts with severity HIGH, require admin approval

### Task 4.3: Report Generation
- Multi-format: Markdown (human), JSON (API), CSV (audit)
- Summary: total items captured, lost, modified, preserved per category
- Risk score: % of items lost (0% = perfect preservation, >50% = critical loss)

## Phase 5 — External Backup Module

### Task 5.1: REST Export Client
- Configurable endpoint URL, API key, timeout
- Retry with exponential backoff (3 attempts)
- Checksum verification: POST includes SHA-256 header, endpoint must echo it back

### Task 5.2: S3 Backup (Optional)
- Alternative backup target via S3-compatible API
- AWS SigV4 signing
- Bucket/key configuration in shield config

## Phase 6 — Restore Engine

### Task 6.1: Update Set Restore
- Re-import update set XML from stored snapshot
- Create new update set record with `[RESTORED]` prefix + original name + timestamp
- Conflict handling: if same-named update set exists, require admin override

### Task 6.2: System Property Restore
- Restore overridden system properties
- Skip redacted properties (credentials must be re-entered manually)
- Log all restored properties

### Task 6.3: Custom Code Restore
- Restore Script Includes, Business Rules, UI Policies, UI Actions
- Warn if target record already exists (possible merge conflict)
- Dry-run mode: preview restores without executing

## Phase 7 — Testing & QA

### Task 7.1: Snapshot Integrity Tests
- Full snapshot on instance with 100+ update sets, 50+ custom properties
- Verify all items captured, checksum matches
- Verify external export succeeded

### Task 7.2: Delta Accuracy Tests
- Create pre-snapshot baseline
- Simulate clone: delete 10 items, modify 5, add 3 new
- Verify delta correctly classifies all changes

### Task 7.3: Restore Tests
- Delete 5 update sets from post-clone instance
- Run restore from pre-snapshot
- Verify all 5 update sets re-created with correct XML content

### Task 7.4: Edge Cases
- Empty sandbox (0 items) → snapshot creates empty but valid JSON
- Maximum sandbox (10K+ items) → chunked capture, no timeout
- External endpoint offline → attachment fallback, warning logged
- Credential redaction → verify no passwords/keys in snapshot
- Cross-instance restore attempted → shield blocks with identity mismatch warning

## Dependencies & Blockers

| Blocker | Status | Resolution |
|---------|--------|------------|
| External REST endpoint for snapshot backup | ⚠️ Pending | User provides endpoint or uses attachment-only mode |
| Cross-scope privilege grants | ⚠️ Pending | Must be granted by admin on sandbox instance |
| Attachment quota monitoring | ✅ Resolved | sys_attachment table query available |

## Rollback Plan

If restore introduces conflicts:
1. Mark restored items as `[CONFLICT]` status in shield audit table
2. Do not auto-apply conflicted restores — require manual resolution
3. Keep pre-snapshot intact — restore is idempotent, can be retried
4. Audit trail shows exactly what was restored and when
