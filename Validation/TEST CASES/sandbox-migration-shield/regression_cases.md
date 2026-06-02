# Sandbox Migration Shield — Regression Cases

**Date:** 2026-06-01  
**Author:** Vladimir Kapustin

## REG-01: Snapshot JSON Schema Stability
**Purpose:** Snapshot JSON format must not change between versions — downstream parsers break  
**Input:** Snapshot from v1.0  
**Expected:** v1.1 produces identical JSON structure (same keys, same types)  
**Verification:** Diff snapshot JSON schemas between versions  
**Fail if:** New mandatory field added, existing field type changed, existing field removed

## REG-02: GlideEncrypter Round-Trip After Instance Restart
**Purpose:** Encrypted snapshots must remain decryptable after instance restart/wake  
**Input:** Stored encrypted snapshot  
**Expected:** Decrypt succeeds after simulated restart  
**Verification:** Store snapshot → simulate restart (re-read from DB) → decrypt → compare  
**Fail if:** Decryption fails or content differs from original

## REG-03: Update Set XML Fidelity
**Purpose:** Restored update set XML must be byte-identical to original  
**Input:** Update set "My Feature" with 5 update_xml records  
**Expected:** After snapshot + restore, all 5 XML records match original  
**Verification:** Compare original XML vs restored XML character-by-character  
**Fail if:** Any XML record differs (whitespace change, encoding shift)

## REG-04: System Property Type Preservation
**Purpose:** Restored system properties must retain their type (string, integer, boolean, choice list)  
**Input:** Properties: stringProp="hello", intProp="42", boolProp="true", choiceProp="option_a"  
**Expected:** After restore, type = string/int/boolean/choice respectively  
**Verification:** Query sys_dictionary for restored properties, verify type field  
**Fail if:** intProp becomes string, boolProp becomes string

## REG-05: Scheduled Job Cron Expression Preservation
**Purpose:** Restored scheduled jobs must keep exact cron expressions  
**Input:** Job with cron `0 0 2 * * ?`  
**Expected:** After restore, cron = `0 0 2 * * ?`  
**Verification:** Query sysauto_script for restored job, compare cron_schedule  
**Fail if:** Cron expression altered, timezone shifted, or schedule changed

## REG-06: Attachment Quota Handling
**Purpose:** Multiple snapshot attachments must not silently fail when quota reached  
**Input:** 5 snapshots, attachment quota = 3  
**Expected:** Oldest 2 snapshots auto-purged, newest 3 preserved  
**Verification:** Query sys_attachment for snapshot files, verify count = 3  
**Fail if:** 4th snapshot fails without purge, 5 snapshots exist exceeding quota

## REG-07: External Endpoint Authentication Persistence
**Purpose:** External backup API key must survive snapshot config updates  
**Input:** API key = "sk-test-12345" stored in config  
**Expected:** After multiple snapshot runs, API key unchanged, all exports authenticated  
**Verification:** Query shield config after 5 snapshot runs, verify API key matches original  
**Fail if:** API key truncated, encrypted differently, or missing

## REG-08: Concurrent Snapshot Serialization
**Purpose:** Two simultaneous snapshot requests must serialize, not corrupt data  
**Input:** Trigger snapshots A and B simultaneously for same instance  
**Expected:** A completes fully, then B starts, each gets a clean snapshot  
**Verification:** Check snapshots A and B — both complete, no interleaved data  
**Fail if:** B overwrites A's snapshot record mid-write, corrupted JSON in either

## REG-09: Unicode Character Preservation
**Purpose:** Non-ASCII characters in update set names, property values, script comments must survive round-trip  
**Input:** Update set "機能開発", property value "café", comment "// résumé logic"  
**Expected:** After snapshot + restore, all Unicode characters preserved  
**Verification:** Compare original vs restored for each Unicode-containing string  
**Fail if:** Any character replaced with `?` or garbled

## REG-10: Delta Performance (1K Items < 10s)
**Purpose:** Delta computation must not degrade with each version  
**Input:** Pre-snapshot with 1,000 items, post-scan with 1,000 items (50 changes)  
**Expected:** Delta computation completes in < 10 seconds  
**Verification:** Time delta execution, assert < 10s  
**Fail if:** > 10 seconds (O(n²) regression) or > 30 seconds (severe degradation)
