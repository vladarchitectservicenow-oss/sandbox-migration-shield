# Sandbox Migration Shield — Test Suite SOP

**Date:** 2026-06-01  
**Author:** Vladimir Kapustin  
**Test Environment:** ServiceNow PDI (dev362840) + Python Mock CI

## Pre-Test Setup

1. Verify PDI is active: `https://dev362840.service-now.com`
2. Install shield application via Studio XML import
3. Grant cross-scope read privileges for all tracked tables (see dependency report)
4. Configure external REST endpoint (mock server or httpbin)
5. Create test data: 5 update sets (3 in-progress, 2 complete), 10 system properties, 3 Script Includes, 2 REST endpoints

## Test Scenarios

### SC-01: Full Snapshot — All Categories Captured (PASS)
**Precondition:** Instance has update sets, system props, script includes, REST endpoints, scheduled jobs, email configs  
**Steps:**
1. Run `SandboxSnapshot.collect()` from Background Script
2. Verify snapshot record created in `x_sandbox_migration_shield_snapshot`
3. Decrypt snapshot JSON and verify all categories present
**Expected:** Snapshot contains all categories; 5 update sets, 10+ properties, 3 script includes, 2 REST endpoints  
**Pass Criteria:** All categories non-empty, snapshot_checksum present, no errors

### SC-02: Credential Redaction — Passwords/Keys Hidden
**Precondition:** System properties include `glide.email.smtp.password`, `x_myapp.api_key`, `integration.token`  
**Steps:**
1. Run snapshot with credential detection enabled
2. Decrypt snapshot and check redacted properties
**Expected:** Values for password/key/token properties show `[REDACTED]`  
**Pass Criteria:** At least 3 properties show `[REDACTED]`, all other properties show real values

### SC-03: External Backup — REST Export Success
**Precondition:** External REST endpoint configured and reachable  
**Steps:**
1. Run snapshot with external backup enabled
2. Check shield audit log for export status
**Expected:** Audit log shows "EXPORT_SUCCESS", HTTP 200 from external endpoint  
**Pass Criteria:** Export status = SUCCESS, checksum verified by endpoint

### SC-04: External Backup — REST Endpoint Offline (Fallback)
**Precondition:** External REST endpoint unreachable  
**Steps:**
1. Configure endpoint to unreachable URL
2. Run snapshot with external backup enabled
**Expected:** Audit log shows "EXPORT_FAILED", attachment backup succeeds, snapshot still stored  
**Pass Criteria:** Export status = FAILED, snapshot.status = STORED_ATTACHMENT_ONLY, no data loss

### SC-05: Delta — Added/Deleted/Modified/Preserved Classification
**Precondition:** Pre-snapshot stored  
**Steps:**
1. Store pre-snapshot with 10 items across categories
2. Modify 3 items, delete 2, add 2 new, leave 3 unchanged
3. Run PostRefreshDelta against pre-snapshot
**Expected:** Delta report: 2 ADDED, 2 DELETED, 3 MODIFIED, 3 PRESERVED — total 10  
**Pass Criteria:** All 10 items correctly classified, no false classifications

### SC-06: Delta — Fresh Clone Detection (>90% DELETED)
**Precondition:** Post-clone instance is a fresh copy with 0 customizations  
**Steps:**
1. Store pre-snapshot with 50 custom items
2. Simulate fresh clone (0 items in post)
3. Run delta
**Expected:** Delta shows >90% DELETED, shield flags "fresh clone detected, all items expected"  
**Pass Criteria:** Warning message present, report not cluttered with 50 individual DELETED items

### SC-07: Restore — Update Set Re-import
**Precondition:** Post-clone missing an update set present in pre-snapshot  
**Steps:**
1. Delete update set "My In-Progress Feature" from post-clone
2. Run restore for that update set from pre-snapshot
3. Verify new update set created
**Expected:** New update set created with name `[RESTORED] My In-Progress Feature - 2026-06-01`  
**Pass Criteria:** Update set exists, XML content matches pre-snapshot, status = Complete

### SC-08: Restore — Conflict Detection (Same-Name Update Set)
**Precondition:** Post-clone has update set with same name as pre-snapshot item  
**Steps:**
1. Create update set "My Feature" in post-clone
2. Attempt restore of "My Feature" from pre-snapshot
**Expected:** Shield detects conflict, shows "Name collision: update set 'My Feature' already exists"  
**Pass Criteria:** Restore blocked with CONFLICT status, admin approval required, no duplicate created

### SC-09: Restore — Dry Run Mode
**Steps:**
1. Run restore with `dry_run=true`
2. Verify no actual changes made to instance
**Expected:** Report shows "Would restore: 5 update sets, 10 properties" but 0 items actually restored  
**Pass Criteria:** Dry run report accurate, 0 actual modifications, audit log shows "DRY_RUN"

### SC-10: Large Instance — Chunked Snapshot (10K+ Items)
**Precondition:** 10,000+ items across all categories  
**Steps:**
1. Configure chunk_size=500
2. Run snapshot
3. Verify all items captured
**Expected:** 20 chunks processed, total items = 10,000+, no timeout  
**Pass Criteria:** Chunk count > 1, total items = expected, snapshot complete within 120s

### SC-11: Empty Instance Snapshot
**Precondition:** Fresh PDI with 0 custom items  
**Steps:**
1. Run snapshot
2. Verify snapshot record created
**Expected:** Snapshot with empty categories, no error, status = COMPLETE  
**Pass Criteria:** Snapshot record exists, all category arrays empty, no crash

### SC-12: Snapshot Integrity — Checksum Verification
**Steps:**
1. Run snapshot
2. Compute SHA-256 of snapshot JSON
3. Compare against stored checksum
**Expected:** Checksums match  
**Pass Criteria:** SHA-256 match, no corruption detected

### SC-13: Cross-Instance Restore Prevention
**Precondition:** Pre-snapshot from instance A, attempting restore on instance B  
**Steps:**
1. Load snapshot with instance_id = "A"
2. Attempt restore on instance B
**Expected:** Shield blocks restore: "Snapshot from instance A cannot be restored to instance B"  
**Pass Criteria:** Restore blocked, error message includes instance IDs

### SC-14: Audit Trail Completeness
**Steps:**
1. Run snapshot → delta → restore cycle
2. Query shield audit table
**Expected:** 3 audit records: SNAPSHOT_CREATED, DELTA_COMPUTED, RESTORE_EXECUTED  
**Pass Criteria:** All operations logged with timestamp, user, instance_id, operation_type

## Execution Order

1. SC-01 (baseline snapshot) → SC-05 (delta) → SC-07 (restore)
2. SC-02 (credential redaction)
3. SC-03, SC-04 (external backup)
4. SC-06 (fresh clone detection)
5. SC-08, SC-09 (conflict + dry run)
6. SC-10, SC-11 (large + empty)
7. SC-12, SC-13, SC-14 (integrity + cross-instance + audit)
