# Lakewatch OCSF gold tables — quick reference

Lakewatch normalizes events to a **modified OCSF 1.3.0** schema (small Lakewatch-specific
additions on top of the standard classes — e.g. `lw_id`, `unmapped`, and a few enrichments).
Gold tables live under `<catalog>.gold.*`. The **catalog name varies by environment** — common
patterns include `lw-prod-catalog`, `lakewatch_<env>`, or a sandbox name. Confirm with the user
(or `SHOW CATALOGS`) which catalog their workspace uses; do not hardcode `lw-prod-catalog` into
shipped queries.

This is a quick index. For the full per-column schema, either:

- `DESCRIBE EXTENDED <catalog>.gold.<table>` against the user's workspace, or
- if available locally, read
  `/Users/alexey.ott/work/antimatter/content-marketplace/.agents/skills/lakewatch-rule-dev/references/gold_tables.md`
  (the authoritative per-column listing).

## Available gold tables

| Table | OCSF class_uid | What it holds | Common KQL sources |
| --- | --- | --- | --- |
| `account_change` | 3001 | User account create/update/delete, role assignments | `AuditLogs` (UserManagement), `IdentityDirectoryEvents`, Windows 4720/4722/4723/4724/4725/4726/4738 |
| `api_activity` | 6003 | Control-plane / API calls | `AzureActivity`, CloudTrail equivalents |
| `authentication` | 3002 | Sign-ins, logons, MFA challenges | `SigninLogs`, `AAD*SignInLogs`, `DeviceLogonEvents`, `IdentityLogonEvents`, Windows 4624/4625 |
| `data_security_finding` | 2006 | DLP-style findings | DLP products |
| `datastore_activity` | 6004 | Database/storage access | DB audit logs |
| `detection_finding` | 2004 | Vendor alerts / detections | `SecurityAlert`, `AlertInfo`, `AlertEvidence` |
| `dhcp_activity` | 4004 | DHCP lease events | DHCP logs |
| `dns_activity` | 4003 | DNS queries / responses | Zeek dns, Defender DNS, Windows DNS |
| `email_activity` | 4009 | Email send/receive/quarantine | `EmailEvents`, M365 mail trace |
| `entity_management` | 3004 | Generic create/update/delete on entities (incl. registry) | `DeviceRegistryEvents`, `DeviceEvents` |
| `file_activity` | 1001 | File create/modify/delete/rename | `DeviceFileEvents`, Windows 5140/5145, OneDrive |
| `file_hosting_activity` | 6006 | Cloud file-share activity | SharePoint, OneDrive admin |
| `group_management` | 3006 | Group create / member add/remove | `AuditLogs` (GroupManagement), Windows 4727…4735 |
| `http_activity` | 4002 | HTTP / URL clicks | `UrlClickEvents`, proxy logs |
| `incident_finding` | 2005 | Incident-level findings | `SecurityIncident` |
| `network_activity` | 4001 | TCP/UDP connection events | `DeviceNetworkEvents`, VNet Flow, Firewall, Zeek conn |
| `process_activity` | 1007 | Process create / terminate / inject | `DeviceProcessEvents`, Windows 4688 |
| `scheduled_job_activity` | 5003 | Scheduled task / cron events | Windows Task Scheduler, cron |
| `script_activity` | 1009 | Script execution | PowerShell, AMSI |
| `ssh_activity` | 4007 | SSH session events | SSHD logs |
| `user_access` | 3005 | Permission grants / RBAC changes | Various IAM sources |
| `vulnerability_finding` | 2002 | Vuln scan findings | Tenable, Qualys, Defender VM |

## Universal columns (every gold table has these)

| Column | Type | What it holds |
| --- | --- | --- |
| `lw_id` | `string` | Lakewatch lineage ID (don't filter on this) |
| `time` | `timestamp` | Event time. **Always filter on this** for performance. |
| `activity_id`, `activity_name`, `activity` | `int` / `string` | Normalized activity (e.g., `Launch`, `Create`, `Logon`) |
| `category_uid`, `category_name` | `int` / `string` | OCSF category |
| `class_uid`, `class_name` | `int` / `string` | OCSF class — same as the table identity |
| `severity_id`, `severity` | `int` / `string` | 1=Informational, 2=Low, 3=Medium, 4=High, 5=Critical, 6=Fatal |
| `status_id`, `status`, `status_detail`, `status_code` | int/string/string/string | 1=Success, 2=Failure, 3=Unknown |
| `type_uid`, `type_name` | `bigint` / `string` | `class_uid * 100 + activity_id` |
| `metadata` | `struct` | Product / log / processing metadata — see below |
| `unmapped` | `variant` | Vendor-specific fields that don't fit OCSF |
| `observables` | `array<struct>` | Indicators of interest extracted from the event |
| `raw_data` | `variant` | Original silver `data` payload (when preserved) |

## `metadata` struct

| Path | Holds |
| --- | --- |
| `metadata.product.name` | Product the event came from (e.g., `Microsoft Defender for Endpoint`, `Azure Active Directory`, `Microsoft-Windows-Security-Auditing`) |
| `metadata.product.vendor_name` | Vendor (e.g., `Microsoft`) |
| `metadata.product.feature.name` | Optional feature within a product |
| `metadata.uid` | Source-specific event ID (e.g., Defender `ReportId`, Sentinel `_ResourceId`) |
| `metadata.logged_time` | Time the source logged the event |
| `metadata.processed_time` | Time Lakewatch processed it |
| `metadata.log_name` | Lakewatch log family (e.g., `defender_xdr`, `azure_aad_signins`) |
| `metadata.log_provider` | Logical provider (e.g., `microsoft`, `aws`) |
| `metadata.tenant_uid` | Tenant identifier when applicable |
| `metadata.event_code` | Source action code (e.g., Defender `ActionType`, Windows `EventID`) |

### Picking the right source filter

When a gold table holds events from multiple presets, you need a filter. **Prefer `metadata.log_name`
over `metadata.product.name`** — `log_name` is set from the preset's `sourceType` (short strings
like `defender_xdr`, `microsoft_entra_authentication`, `azure_activity_logs`), so it's smaller on
disk, gets better Delta data-skipping, and produces a faster `WHERE`. `metadata.log_provider` is
the next step up (preset's `source` — `microsoft`, `aws`, …) and is useful when you want every
event from a vendor regardless of product.

Rule of thumb:

1. **`metadata.log_name`** — primary filter. Use this whenever you can.
2. **`metadata.log_name` + `metadata.log_provider`** — when you want to belt-and-suspenders the
   vendor cut.
3. **`metadata.product.name`** — only when `log_name` is too coarse for the discrimination you
   need. The biggest example: on `authentication`, Defender XDR writes **both** `DeviceLogonEvents`
   (Endpoint) and `IdentityLogonEvents` (Identity) with `log_name = 'defender_xdr'` — to isolate
   one product within that family, add `metadata.product.name = 'Microsoft Defender for Endpoint'`
   (or `… for Identity`).

## Common entity sub-structs

Most gold tables include some subset of these. Always use the struct path (`user.name`), never
backticks (`\`user.name\``).

- **`actor`** — who initiated the activity. Common paths: `actor.user.name`, `actor.user.uid`,
  `actor.process.name`, `actor.process.pid`, `actor.process.cmd_line`, `actor.app_name`.
- **`user`** — target user (for authentication / account_change / etc.). Paths: `user.name`,
  `user.uid`, `user.type`, `user.account.name`.
- **`device`** — host context. `device.hostname`, `device.uid`, `device.os.name`.
- **`src_endpoint`** / **`dst_endpoint`** — network endpoints. `*.ip`, `*.port`, `*.hostname`,
  `*.location.country`, `*.location.city`.
- **`process`** — for `process_activity` and `script_activity`. `process.cmd_line`, `process.pid`,
  `process.name`, `process.file.path`, `process.file.hashes` (array).
- **`file`** — for `file_activity`. `file.name`, `file.path`, `file.hashes`.
- **`email`** — for `email_activity`. `email.from`, `email.to` (array), `email.subject`,
  `email.message_uid`.
- **`url`** — for `http_activity`. `url.url_string`, `url.hostname`, `url.path`.
- **`finding_info`** — for `detection_finding` / `incident_finding`. `finding_info.title`,
  `finding_info.desc`, `finding_info.types`.

## Hashes pattern

Hashes are arrays of structs, not flat columns:

```sql
-- Find a process by SHA256
SELECT *
FROM gold.process_activity
WHERE EXISTS (
  SELECT 1 FROM EXPLODE(process.file.hashes) AS h
  WHERE h.algorithm = 'SHA-256' AND h.value = '<sha256>'
)
```

Or, more concisely with `array_contains` + `struct`:

```sql
WHERE array_contains(process.file.hashes,
  named_struct('algorithm', 'SHA-256', 'algorithm_id', 3, 'value', '<sha256>'))
```

## OCSF reference

Authoritative class schemas: <https://schema.ocsf.io/1.3.0/classes/> (Lakewatch is on a modified
1.3.0 — most paths match the standard, but expect a handful of Lakewatch-specific additions like
`lw_id` and `unmapped`). If you're unsure about a column path, check the OCSF doc for the matching
class before guessing.
