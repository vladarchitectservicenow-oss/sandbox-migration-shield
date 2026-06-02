# Sandbox Migration Shield — Edge Cases

**Date:** 2026-06-01  
**Author:** Vladimir Kapustin

## EC-01: Snapshot on Hibernating Instance
**Condition:** Instance enters hibernation mid-snapshot  
**Expected Behavior:** Current chunk fails, snapshot marked as PARTIAL with progress saved, resumes from last chunk on wake  
**Risk:** Corrupted partial snapshot, lost progress  
**Test:** Simulate hibernation (kill mid-snapshot process), verify partial marker and resume capability

## EC-02: Update Set with Circular Dependency
**Condition:** Update set A depends on update set B, which depends on A  
**Expected Behavior:** Both captured, dependency noted in snapshot, restore handles circular deps gracefully (restore both or neither)  
**Risk:** Infinite recursion during restore  
**Test:** Create circular dependency, snapshot + restore

## EC-03: System Property with NULL Value
**Condition:** Property exists in sys_properties but value is NULL  
**Expected Behavior:** Property captured with value=NULL, not skipped  
**Risk:** NULL value treated as missing property, skipped in snapshot  
**Test:** Mock property with NULL value, verify it appears in snapshot

## EC-04: REST Endpoint with Custom Authentication Header
**Condition:** REST endpoint uses `X-Custom-Auth` header instead of standard Basic/Bearer  
**Expected Behavior:** Snapshot captures full header configuration, restore recreates it  
**Risk:** Custom header lost, endpoint fails after restore  
**Test:** Create endpoint with custom auth header, snapshot + restore + verify

## EC-05: Script Include > 1MB
**Condition:** Single Script Include source code exceeds 1MB  
**Expected Behavior:** Captured in full in snapshot, chunk handling if needed for attachment  
**Risk:** Truncation, OutOfMemory on GlideRecord.getString()  
**Test:** Mock 1.5MB script include, verify full capture

## EC-06: Email Configuration with Invalid SMTP Server
**Condition:** Email config references SMTP server that no longer exists  
**Expected Behavior:** Config captured as-is, warning logged during snapshot ("SMTP server unreachable") but capture continues  
**Risk:** Snapshot aborts on connectivity check  
**Test:** Configure email with unreachable SMTP, run snapshot

## EC-07: Update Set with 0 Update XML Records
**Condition:** Update set exists but has 0 sys_update_xml children (empty set)  
**Expected Behavior:** Update set captured, marked as "empty", no restore error  
**Risk:** Empty update set causes NULL pointer or division by zero in stats  
**Test:** Create empty update set, snapshot + delta + restore

## EC-08: Instance Has Duplicate Update Set Names
**Condition:** Two update sets named "My Feature" (different sys_ids)  
**Expected Behavior:** Both captured, both restorable, restore handles collision by appending sys_id suffix  
**Risk:** Name collision during restore, one silently overwrites the other  
**Test:** Create two same-named update sets, restore both

## EC-09: External Endpoint Returns HTTP 500 Mid-Export
**Condition:** Export starts (202 Accepted), then endpoint crashes during transfer  
**Expected Behavior:** Export retries 3 times with backoff, marks FAILED if all retries fail, attachment backup preserved  
**Risk:** Partial upload left on endpoint, attachment deleted prematurely  
**Test:** Simulate endpoint returning 500 after 50% upload

## EC-10: Snapshot JSON Exceeds Maximum Attachment Size
**Condition:** Snapshot JSON > 500MB (ServiceNow attachment limit)  
**Expected Behavior:** Chunked attachment: split into N attachments (part_1, part_2, ...), metadata records part count  
**Risk:** Attachment upload fails with "size exceeded", snapshot lost  
**Test:** Generate 600MB snapshot, verify chunked upload

## EC-11: Post-Clone Instance Has Different sys_id for Same Named Record
**Condition:** Update set "My Feature" has sys_id X in pre-snapshot, but after clone it has sys_id Y (same name)  
**Expected Behavior:** Delta detects as MODIFIED (not ADDED), matches by name+category when sys_id differs  
**Risk:** False ADDED + DELETED instead of MODIFIED, doubling the apparent change volume  
**Test:** Pre/post with different sys_ids for same logical item

## EC-12: Restore Attempt with Corrupted Snapshot (Checksum Mismatch)
**Condition:** Snapshot stored with checksum ABC, but restored snapshot has checksum XYZ  
**Expected Behavior:** Shield refuses restore, shows "Snapshot integrity check failed: checksum mismatch"  
**Risk:** Corrupted data restored, causing production issues  
**Test:** Tamper with stored snapshot, attempt restore, verify block
