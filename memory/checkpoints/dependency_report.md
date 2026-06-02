# Sandbox Migration Shield — Dependency Report

**Date:** 2026-06-01  
**Author:** Vladimir Kapustin

## Internal Dependencies (ServiceNow Platform)

| Dependency | Type | Required Version | Purpose |
|-----------|------|-----------------|---------|
| GlideRecord API | Platform API | All versions | Snapshot collection and delta queries |
| GlideEncrypter | Platform API | All versions | Snapshot data encryption at rest |
| sys_update_set | Platform Table | All versions | Update set capture and restore |
| sys_update_xml | Platform Table | All versions | Update set XML storage and re-import |
| sys_properties | Platform Table | All versions | System property capture |
| sys_dictionary | Platform Table | All versions | Custom table/field detection |
| sys_script_include | Platform Table | All versions | Script Include capture |
| sys_rest_message | Platform Table | All versions | REST endpoint capture |
| sys_attachment | Platform Table | All versions | Snapshot export as attachment |
| sys_scope | Platform Table | All versions | Cross-scope application detection |
| sys_email | Platform Table | All versions | Email configuration capture |

## External Dependencies

| Dependency | Type | Version | Purpose |
|-----------|------|---------|---------|
| External REST Endpoint | HTTP/HTTPS | Any | Off-instance snapshot backup storage |
| S3-compatible Storage | HTTP API | — | Alternative backup target (optional) |

## Plugin Dependencies

None. The shield operates on core platform tables only — no plugins required.

## Cross-Scope Access Requirements

The shield's scope (`x_sandbox_migration_shield`) requires read access to:

| Table | Scope | Purpose |
|-------|-------|---------|
| sys_update_set | Global | Read in-progress update sets |
| sys_update_xml | Global | Read and store update set XML |
| sys_properties | Global | Read system properties |
| sys_dictionary | Global | Detect custom tables/fields |
| sys_script_include | Global | Read Script Includes |
| sys_rest_message | Global | Read REST endpoint configs |
| sys_attachment | Global | Create snapshot attachments |
| sys_scope | Global | Identify scoped applications |
| sys_email | Global | Read email configurations |
| sysauto_script | Global | Read scheduled jobs |
| sys_ui_policy | Global | Read UI policies |
| sys_ui_action | Global | Read UI actions |

## Compatibility Matrix

| Release | Supported | Notes |
|---------|-----------|-------|
| Washington DC | ✅ | Minimum version — snapshot APIs stable |
| Vancouver | ✅ | All core tables available |
| Yokohama | ✅ | Full support |
| Zurich | ✅ | Full support |
| Australia | ✅ | Target release — optimal encryption APIs |

## Risk of Dependency Breakage

- **Low:** Core platform tables (sys_update_set, sys_properties, sys_script_include) are stable across releases
- **Low:** GlideEncrypter — core platform API, no schema changes expected
- **Medium:** External REST endpoint availability — if external storage is down, snapshot is lost when sandbox is wiped. Mitigation: dual backup (attachment + REST).
- **Low:** sys_attachment — size limits may apply; snapshots over 500MB should use chunked upload.

## Mitigation

1. Dual backup: attachment (in-instance) + REST (external) for every snapshot
2. Chunked attachment upload for snapshots >100MB
3. External endpoint health check before snapshot — warn if unreachable
4. Retention policy: auto-purge snapshots older than 90 days to control storage
