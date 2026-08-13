# Defender XDR Advanced Hunting → Lakewatch tables

This file maps **Microsoft Defender XDR Advanced Hunting** tables (the ones you query in the
M365 Defender / Defender Advanced Hunting portal) to Lakewatch tables, plus the most common
column-level mappings.

> **Verify before relying.** The column mappings below are extracted from the
> `microsoft_defender_xdr` preset in the antimatter content-marketplace repo. The Defender preset
> is **selective** about which fields it promotes to typed OCSF paths — many fields that look like
> they "should" be on a struct path are actually under `unmapped` (a VARIANT column), and OCSF 1.3.0
> doesn't expose every field people expect (no `email.from`, no `email.subject`, no `email.message_uid`,
> no `threat.type` array on `email_activity`). When in doubt, `DESCRIBE EXTENDED <catalog>.gold.<table>`
> or read the preset YAML directly at `packs/microsoft/presets/microsoft/defender_xdr/preset.yaml`
> in that repo, rather than inferring from OCSF docs.

## How Lakewatch ingests Defender

Lakewatch's `microsoft_defender_xdr` preset reads Defender events from Azure Event Hub (JSONL),
lands them in **bronze** as a VARIANT column called `data`, routes them by `category` into
per-domain **silver** tables (`mde_*`), then normalizes to **OCSF gold** tables.

| Silver table | Holds Advanced Hunting tables |
| --- | --- |
| `mde_device_events` | `DeviceProcessEvents`, `DeviceFileEvents`, `DeviceNetworkEvents`, `DeviceRegistryEvents`, `DeviceEvents`, `DeviceImageLoadEvents`, `DeviceLogonEvents` |
| `mde_device_info` | `DeviceInfo` (inventory snapshot) |
| `mde_device_network_info` | `DeviceNetworkInfo` (inventory snapshot) |
| `mde_device_file_certificate_info` | `DeviceFileCertificateInfo` |
| `mde_identity_events` | `IdentityLogonEvents`, `IdentityDirectoryEvents` |
| `mde_alert_events` | `AlertInfo`, `AlertEvidence` |
| `mde_email_events` | `EmailEvents` |
| `mde_url_events` | `UrlClickEvents` |

`CloudAppEvents` is **not currently ingested**. `IdentityInfo`, `IdentityQueryEvents`,
`EmailAttachmentInfo`, `EmailUrlInfo` are ingested into silver but **not yet mapped to gold** — use
the silver table for those.

## Source → destination table table

Prefer the OCSF gold table when one exists. Fall back to silver for inventory snapshots and the
not-yet-mapped tables above.

**Filter strategy** — every gold row from the Defender preset carries
`metadata.log_name = 'defender_xdr'` and `metadata.log_provider = 'microsoft'`. Prefer those for
the source filter (shorter values, better Delta data-skipping). Add `metadata.product.name` only
when you need to discriminate between Defender products **within** the same gold table — that
mostly matters on `authentication`, where Endpoint (`DeviceLogonEvents`) and Identity
(`IdentityLogonEvents`) coexist.

| Defender table | OCSF gold (preferred) | OCSF class_uid | Recommended filter (fast → precise) |
| --- | --- | --- | --- |
| `DeviceProcessEvents` | `process_activity` | 1007 | `metadata.log_name = 'defender_xdr'` |
| `DeviceFileEvents` | `file_activity` | 1001 | `metadata.log_name = 'defender_xdr'` |
| `DeviceNetworkEvents` | `network_activity` | 4001 | `metadata.log_name = 'defender_xdr'` |
| `DeviceLogonEvents` | `authentication` | 3002 | `metadata.log_name = 'defender_xdr' AND metadata.product.name = 'Microsoft Defender for Endpoint'` (the `product.name` is needed because Defender for Identity also writes here) |
| `DeviceRegistryEvents` | `entity_management` | 3004 | `metadata.log_name = 'defender_xdr'` (optionally `AND metadata.event_code LIKE 'RegistryValue%'` to narrow by action) |
| `DeviceEvents` (catch-all) | `entity_management` | 3004 | `metadata.log_name = 'defender_xdr'` — `entity.type` discriminates within |
| `DeviceImageLoadEvents` | no gold mapping yet — query `mde_device_events` directly with `event_category = 'DeviceImageLoadEvents'` |  |  |
| `IdentityLogonEvents` | `authentication` | 3002 | `metadata.log_name = 'defender_xdr' AND metadata.product.name = 'Microsoft Defender for Identity'` |
| `IdentityDirectoryEvents` | `account_change` | 3001 | `metadata.log_name = 'defender_xdr'` |
| `AlertInfo` / `AlertEvidence` | `detection_finding` | 2004 | `metadata.log_name = 'defender_xdr'` (Defender alerts dominate this table; product filter only if you also have Sentinel-managed alerts from non-Defender sources) |
| `EmailEvents` | `email_activity` | 4009 | `metadata.log_name = 'defender_xdr'` (currently the only producer of `email_activity`) |
| `UrlClickEvents` | `http_activity` | 4002 | `metadata.log_name = 'defender_xdr'` |
| `DeviceInfo` | (silver only) `mde_device_info` | — | — |
| `DeviceNetworkInfo` | (silver only) `mde_device_network_info` | — | — |
| `DeviceFileCertificateInfo` | (silver only) `mde_device_file_certificate_info` | — | — |
| `IdentityInfo` | (silver only — schema TBD) | — | check `mde_identity_*` |
| `IdentityQueryEvents` | (silver only) `mde_identity_events` filtered by category | — | — |
| `EmailAttachmentInfo`, `EmailUrlInfo` | (silver only) `mde_email_events` filtered by category | — | — |
| `CloudAppEvents` | **not ingested** — stop and tell the user | — | — |

## Column mappings — `DeviceProcessEvents` → `process_activity`

| Defender column | Gold (`process_activity`) |
| --- | --- |
| `Timestamp` | `time` |
| `DeviceId` | `device.uid` |
| `DeviceName` | `device.hostname` |
| `ActionType` | `metadata.event_code` (and drives `activity_id`/`activity_name`) |
| `ProcessCommandLine` | `process.cmd_line` |
| `FileName` | `process.name`, `process.file.name` |
| `FolderPath` | `process.file.path` |
| `ProcessId` | `process.pid` |
| `ProcessUniqueId` | `process.uid` |
| `SHA256`, `SHA1`, `MD5` | inside `process.file.hashes` (array of `{algorithm, algorithm_id, value}`) |
| `AccountName` | `process.user.name` |
| `AccountSid` | `process.user.uid` |
| `AccountDomain` | `process.user.account.name` |
| `InitiatingProcessFileName` | `actor.process.name`, `actor.process.file.name` |
| `InitiatingProcessCommandLine` | `actor.process.cmd_line` |
| `InitiatingProcessId` | `actor.process.pid` |
| `InitiatingProcessFolderPath` | `actor.process.file.path` |
| `InitiatingProcessSHA256`/`SHA1`/`MD5` | inside `actor.process.file.hashes` |
| `InitiatingProcessAccountName` | `actor.user.name` |
| `InitiatingProcessAccountSid` | `actor.user.uid` |
| `InitiatingProcessAccountDomain` | `actor.user.account.name` |
| `InitiatingProcessParentFileName`/`Id` | `actor.process.parent_process` (VARIANT struct with `name`, `pid`) |
| `ReportId` | `metadata.uid` |

Fields not mapped to OCSF (e.g., `ProcessIntegrityLevel`, `InitiatingProcessTokenElevation`,
`InitiatingProcessSignatureStatus`) live inside the `unmapped` VARIANT column on the gold row.
Reach them with `variant_get(unmapped, '$.ProcessIntegrityLevel', 'string')`.

## Column mappings — `DeviceFileEvents` → `file_activity`

| Defender column | Gold (`file_activity`) |
| --- | --- |
| `Timestamp` | `time` |
| `ActionType` | drives `activity_id`/`activity_name` (`FileCreated` → 1 / Create, `FileModified` → 3 / Update, `FileDeleted` → 4 / Delete, `FileRenamed` → 5 / Rename) |
| `FileName` | `file.name` |
| `FolderPath` | `file.path` |
| `SHA256`/`SHA1`/`MD5` | inside `file.hashes` |
| Initiating-process fields | `actor.process.*` (same shape as in `process_activity`) |
| `DeviceId` / `DeviceName` | `device.uid` / `device.hostname` |

## Column mappings — `DeviceNetworkEvents` → `network_activity`

The Defender preset for `DeviceNetworkEvents` is **leaner** than for `DeviceProcessEvents` — only
endpoint/connection fields make it into typed OCSF columns; initiating-process and device-identity
fields land in `unmapped`. This catches people out, so don't assume `actor.process.*` or
`device.*` will be populated here.

| Defender column | Gold (`network_activity`) | Notes |
| --- | --- | --- |
| `Timestamp` | `time` | |
| `ActionType` | `metadata.event_code` | also drives `activity_id` / `status_id` |
| `LocalIP` | `src_endpoint.ip` | |
| `LocalPort` | `src_endpoint.port` | |
| `DeviceName` (the local host) | `src_endpoint.hostname` | the gold preset uses `device_name` from silver |
| `RemoteIP` | `dst_endpoint.ip` | |
| `RemotePort` | `dst_endpoint.port` | |
| `RemoteUrl` | `dst_endpoint.domain` **and** `url.url_string` | preset writes the same value to both |
| `Protocol` | `connection_info.protocol_name` | |
| `LocalIPType`/`RemoteIPType` | drive `connection_info.direction_id` / `direction` | derived (Private→Public → Outbound, etc.) |
| `InitiatingProcessFileName` | `variant_get(unmapped, '$.InitiatingProcessFileName', 'string')` | **NOT `actor.process.*`** — Defender's network mapping puts initiating-process fields in `unmapped` |
| `InitiatingProcessId` | `variant_get(unmapped, '$.InitiatingProcessId', 'string')` | |
| `InitiatingProcessCommandLine` | `variant_get(unmapped, '$.InitiatingProcessCommandLine', 'string')` | |
| `InitiatingProcessAccountName` | `variant_get(unmapped, '$.InitiatingProcessAccountName', 'string')` | |
| `DeviceId` | **not promoted** — read from silver `mde_device_events.device_id`, or `variant_get(raw_data, '$.properties.DeviceId', 'string')` | the `device.uid` field is NOT written for network events |
| `ReportId` | `metadata.uid` | |

## Column mappings — `DeviceLogonEvents` → `authentication`

| Defender column | Gold (`authentication`) |
| --- | --- |
| `Timestamp` | `time` |
| `ActionType` | `metadata.event_code` |
| `LogonType` | `auth_protocol` / `logon_type` (and informs `activity_id`) |
| `AccountName` | `user.name` |
| `AccountDomain` | `user.account.name` |
| `AccountSid` | `user.uid` |
| `RemoteIP` | `src_endpoint.ip` |
| `RemotePort` | `src_endpoint.port` |
| `RemoteDeviceName` | `src_endpoint.hostname` |
| `DeviceId` / `DeviceName` | `dst_endpoint.uid` / `dst_endpoint.hostname` |
| Logon success/failure | `status_id` (1 = Success, 2 = Failure), `status` |

## Column mappings — `EmailEvents` → `email_activity`

OCSF 1.3.0 `email_activity` has a narrower typed schema than people often expect — there is no
`email.from` / `email.subject` / `email.message_uid` / `threat.type` array. The Defender preset
puts most email metadata in `unmapped` and uses a few OCSF top-level fields (`message`,
`message_trace_uid`, `email.to`, `src_endpoint.*`, `direction*`, `disposition*`).

| Defender column | Gold (`email_activity`) | Notes |
| --- | --- | --- |
| `Timestamp` | `time` | |
| `NetworkMessageId` | `message_trace_uid` **and** `metadata.correlation_uid` | the preset writes the same value to both; **NOT** `email.message_uid` |
| `Subject` | `message` | **NOT** `email.subject` (OCSF 1.3 puts it on `message`) |
| `RecipientEmailAddress` | `email.to` | scalar string in the preset (`email.to` here is not an array on the typed projection) |
| `SenderFromDomain` | `src_endpoint.domain` | |
| `SenderIPv4` / `SenderIPv6` | `src_endpoint.ip` (via `COALESCE`) | |
| `SenderFromAddress` | `variant_get(unmapped, '$.SenderFromAddress', 'string')` | **NOT** `email.from` — it's in `unmapped` |
| `SenderDisplayName` | `variant_get(unmapped, '$.SenderDisplayName', 'string')` | |
| `SenderMailFromAddress` / `SenderMailFromDomain` | `variant_get(unmapped, '$.SenderMailFromAddress' / '.SenderMailFromDomain', 'string')` | |
| `InternetMessageId` | `variant_get(unmapped, '$.InternetMessageId', 'string')` | |
| `DeliveryAction` | drives `activity_id` (Inbound→2/Outbound→1), `status_id` (Delivered→1/Blocked→2), `disposition_id` (Delivered→1/Blocked→2/Junked→3/Replaced→99), plus `metadata.event_code`; raw value is also in `unmapped.DeliveryAction` | |
| `DeliveryLocation` | `variant_get(unmapped, '$.DeliveryLocation', 'string')` | **NOT** `delivery_status` — that OCSF field is not used by this preset |
| `EmailDirection` | drives `direction_id` / `direction`; raw value in `unmapped.EmailDirection` | |
| `ThreatTypes` | `variant_get(unmapped, '$.ThreatTypes', 'string')` | **string**, NOT an array — Defender ships ThreatTypes as a comma-/semicolon-delimited string. KQL `has 'Phish'` becomes `... ILIKE '%Phish%'` (or `regexp_like(..., '(^|[,;\\s])Phish($|[,;\\s])')` for token-strict matching). There is **no `threat.type` array** on this gold table. |
| `ThreatNames`, `DetectionMethods` | `variant_get(unmapped, '$.ThreatNames' / '.DetectionMethods', 'string')` | |
| `ReportId` | `metadata.uid` | |
| `TenantId` | `metadata.tenant_uid` | |

## Column mappings — `UrlClickEvents` → `http_activity`

The Defender URL-click mapping is even more `unmapped`-heavy than email. `Workload`,
`NetworkMessageId`, `AccountUpn`, `ThreatTypes`, and several other fields land in `unmapped` —
**not** in OCSF struct paths.

| Defender column | Gold (`http_activity`) | Notes |
| --- | --- | --- |
| `Timestamp` | `time` | |
| `Url` | `http_request.url` | **NOT** `url.url_string` — the preset writes to `http_request.url` |
| `ActionType` (`ClickAllowed`, `ClickBlocked`, `ClickQuarantined`, …) | `metadata.event_code`, `status_detail` | also drives `status_id`/`severity_id`/`action_id`/`disposition_id` |
| `IPAddress` | `src_endpoint.ip` | |
| `AccountUpn` | `variant_get(unmapped, '$.AccountUpn', 'string')` | **NOT** `actor.user.name` — Defender keeps the UPN in `unmapped` |
| `AppName` | `app_name` (top-level) **and** also `variant_get(unmapped, '$.AppName', 'string')` | |
| `AppVersion` | `variant_get(unmapped, '$.AppVersion', 'string')` | |
| `Workload` (e.g., `'Email'`, `'Teams'`, `'SharePoint'`) | `variant_get(unmapped, '$.Workload', 'string')` | **NOT** `metadata.product.feature.name` — that field is not written by this preset. The KQL `Workload == 'Email'` becomes `variant_get(unmapped, '$.Workload', 'string') = 'Email'` |
| `NetworkMessageId` | `variant_get(unmapped, '$.NetworkMessageId', 'string')` **and** `metadata.correlation_uid` | the preset writes the same value to both — `metadata.correlation_uid` is the friendlier path for joins back to `email_activity.message_trace_uid` |
| `ThreatTypes`, `DetectionMethods`, `UrlChain`, `IsClickedThrough`, `SourceId` | all under `variant_get(unmapped, '$.<FieldName>', 'string')` | |
| `ReportId` | `metadata.uid` | |
| `TenantId` | `metadata.tenant_uid` | |

**Joining `email_activity` ↔ `http_activity` on the message ID**: prefer
`email_activity.metadata.correlation_uid = http_activity.metadata.correlation_uid` (both set from
`NetworkMessageId`) — it avoids two `variant_get` calls compared to using `unmapped` paths
directly, and `metadata.correlation_uid` is a typed column so the join planner sees stats.

## Querying silver directly (when gold doesn't fit)

When you need a Defender column that didn't get a gold mapping, query silver and use
`try_variant_get` against the `data` VARIANT column:

```sql
SELECT
  device_id,
  device_name,
  try_variant_get(data, '$.properties.ProcessIntegrityLevel', 'string') AS integrity_level,
  try_variant_get(data, '$.properties.ProcessTokenElevation', 'string')   AS token_elevation,
  time
FROM `<catalog>`.silver.mde_device_events
WHERE event_category = 'DeviceProcessEvents'
  AND time >= current_timestamp() - INTERVAL 7 DAY
```

Note: the path is `$.properties.<FieldName>` — that's the Event Hub envelope structure Defender
ships, not a Lakewatch invention.

## Catalog and schema

Tables live under `<catalog>.gold.*` and `<catalog>.silver.*`. The catalog name **varies by
environment** — common patterns are `lw-prod-catalog`, `lakewatch_<env>`, or a sandbox name.
Confirm with the user or `SHOW CATALOGS`. Defender's `metadata.log_name` is `'defender_xdr'` for
all event types (Endpoint / Identity / Office 365), and `metadata.log_provider` is `'microsoft'`.
The `metadata.product.name` literals are also stable (`'Microsoft Defender for Endpoint'`,
`'Microsoft Defender for Identity'`, `'Microsoft Defender for Office 365'`) — use them when you
need to discriminate Defender products within a single gold table; otherwise `log_name` alone is
the faster filter.
