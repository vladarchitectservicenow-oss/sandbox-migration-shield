# Sandbox Migration Shield — Risk Report

**Date:** 2026-06-01  
**Author:** Vladimir Kapustin  
**Classification:** Internal — Security-Relevant

## Risk Register

### P0 — Critical (Must Fix Before Production Use)

| ID | Risk | Impact | Likelihood | Mitigation | Owner |
|----|------|--------|------------|------------|-------|
| R01 | Sandbox fully wiped during clone — in-instance snapshot lost | All pre-refresh data unrecoverable; development work permanently lost | Medium | External backup via REST endpoint before clone; automated health check verifies external backup completeness | Dev |
| R02 | Update set restore conflicts with post-clone update sets | Data corruption; merge conflicts in customizations | Medium | Conflict detection before restore; diff view shows conflicts; admin must approve restore for conflicted items | Dev |
| R03 | Snapshot contains sensitive credentials (sys_properties with API keys) | Credential leak if snapshot exported to insecure external storage | High | Credential field detection and redaction in snapshot; GlideEncrypter before external export; property exclusion list | Dev |

### P1 — High

| ID | Risk | Impact | Likelihood | Mitigation | Owner |
|----|------|--------|------------|------------|-------|
| R04 | External REST endpoint unreachable during snapshot export | Snapshot exists only in-instance; lost if sandbox wiped | Medium | Dual backup (attachment + REST); pre-snapshot connectivity test; alert if REST export fails | Dev |
| R05 | Cross-scope privilege revocations on prod clone into sandbox | Shield cannot read required tables after clone, delta fails | Medium | Post-clone privilege check as first step in delta engine; auto-request privilege grants if missing | Dev |
| R06 | Large update set (>10K records) causes snapshot timeout | Incomplete snapshot, silent data loss | Low | Chunked capture with configurable batch size; snapshot progress tracking; resume from last chunk | Dev |

### P2 — Medium

| ID | Risk | Impact | Likelihood | Mitigation | Owner |
|----|------|--------|------------|------------|-------|
| R07 | Instance-specific configuration (email server, MID server) restored into different sandbox | Broken integrations, email loops | Medium | Instance identity detection; tag configuration with source_instance_id; warn on cross-instance restore | Dev |
| R08 | Delta comparison on empty post-clone instance (fresh clone) | All items show as DELETED — noise overwhelms actual losses | High | Post-clone baseline detection; if instance is fresh clone (>90% items shown as DELETED), flag as "fresh clone, all items expected" | Dev |
| R09 | Attachment storage quota exceeded | Cannot store new snapshots; historical snapshots must be purged | Low | Configurable retention policy; auto-purge oldest snapshots when quota > 80%; alert admin | Dev |

### P3 — Low

| ID | Risk | Impact | Likelihood | Mitigation | Owner |
|----|------|--------|------------|------------|-------|
| R10 | Multiple concurrent clone events | Race condition on snapshot write | Low | Singleton per sandbox instance; lock mechanism via config table flag | Dev |
| R11 | Snapshot file corruption during external export | Unrecoverable backup | Low | Checksum (SHA-256) on snapshot before export; verify checksum after re-import | Dev |
| R12 | Post-clone instance in different timezone | Timestamp comparison skew in delta | Low | Store all timestamps in UTC; conversion only for display | Dev |

## Security Review Checklist

- [x] No hardcoded credentials in source code
- [x] Snapshot data encrypted at rest (GlideEncrypter)
- [x] External export uses HTTPS only
- [x] Credential detection and redaction in snapshot (sys_properties with keys, tokens, passwords)
- [x] Audit logging for all snapshot/restore operations
- [x] Role-based access (admin vs viewer)
- [x] GDPR-compliant retention policy (auto-purge 90 days)
- [x] Instance identity tagging prevents cross-instance contamination

## Residual Risk After Mitigation

After all P0 and P1 mitigations:
- **Snapshot loss during wipe:** Acceptable (dual backup: attachment + external REST)
- **Credential leak in snapshot:** Acceptable (redaction + encryption)
- **Update set conflict on restore:** Tolerable (conflict detection + admin approval gate)
- **External endpoint failure:** Tolerable (attachment fallback + alert)

## Sign-off

| Role | Name | Date | Status |
|------|------|------|--------|
| Developer | Vladimir Kapustin | 2026-06-01 | Approved |
| Security Review | (Pending) | — | — |
| QA Lead | (Pending) | — | — |
