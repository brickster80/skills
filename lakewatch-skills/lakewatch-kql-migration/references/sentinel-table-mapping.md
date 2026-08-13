# Microsoft Sentinel → Lakewatch tables

This file maps **Microsoft Sentinel** Log Analytics tables to Lakewatch tables and lists the most
common column-level mappings.

## How Lakewatch ingests Sentinel-equivalent data

Lakewatch does **not** ingest Sentinel's Log Analytics tables directly. Instead, it ingests the
underlying Microsoft sources via dedicated presets, and normalizes them to OCSF:

| Lakewatch preset | What it ingests |
| --- | --- |
| `microsoft_entra` (in the `microsoft` pack) | All Entra ID (Azure AD) log streams: sign-in activity, audit logs, etc. — fed from Azure Monitor / Event Hub. Silver tables: `microsoft_entra_authentication`, `microsoft_entra_account_change`, `microsoft_entra_entity_management`, `microsoft_entra_group_management`, `microsoft_entra_user_access`, `microsoft_entra_api_activity`. |
| `azure_activity_logs` (in the `azure` pack) | Azure Resource Manager activity logs. Silver: `azure_activity_logs`. |
| `azure_aad_user_risk_events` (in the `azure` pack) | Entra Identity Protection user-risk events. Silver: `azure_aad_user_risk_events`. |
| `azure_aad_risky_users` (in the `azure` pack) | Entra Identity Protection risky-users snapshot. Silver: `azure_aad_risky_users`. |
| `azure_firewall` (in the `azure` pack) | Azure Firewall network rule, app rule, DNS proxy logs. Silver: `azure_firewall`. |
| `azure_vnet_flow_logs` (in the `azure` pack) | Azure VNet Flow Logs (Network Watcher v2). Silver: `azure_vnet_flow_logs`. |
| `microsoft_defender_xdr` (in the `microsoft` pack) | M365 Defender Advanced Hunting tables. See `defender-table-mapping.md` for the breakdown. |
| `windows_events_xml` (in the `microsoft` pack) | Windows Event Log via AMA. Silver: typically a per-source silver table; gold per EventID. |

When a user hands you a Sentinel query, your job is to figure out which underlying source the
Sentinel table is backed by, then map that to the corresponding Lakewatch silver/gold table.

## Source → destination table table

| Sentinel table | Lakewatch preset | Silver table | Gold table (preferred) | OCSF class_uid |
| --- | --- | --- | --- | --- |
| `SigninLogs` | `microsoft_entra` | `microsoft_entra_authentication` | `authentication` | 3002 |
| `AADNonInteractiveUserSignInLogs` | `microsoft_entra` | `microsoft_entra_authentication` (category discriminates) | `authentication` | 3002 |
| `AADServicePrincipalSignInLogs` | `microsoft_entra` | `microsoft_entra_authentication` | `authentication` | 3002 |
| `AuditLogs` (UserManagement category) | `microsoft_entra` | `microsoft_entra_account_change` | `account_change` | 3001 |
| `AuditLogs` (GroupManagement category) | `microsoft_entra` | `microsoft_entra_group_management` | `group_management` | 3006 |
| `AuditLogs` (ApplicationManagement, ServicePrincipal, etc.) | `microsoft_entra` | `microsoft_entra_entity_management` | `entity_management` | 3004 |
| `AuditLogs` (RoleManagement / DirectoryRole assignments) | `microsoft_entra` | `microsoft_entra_user_access` | `user_access` | 3005 |
| `MicrosoftGraphActivityLogs` | `microsoft_entra` | `microsoft_entra_api_activity` | `api_activity` | 6003 |
| `AADUserRiskEvents` | `azure_aad_user_risk_events` | `azure_aad_user_risk_events` | `detection_finding` (primary), also routes to `authentication` and `account_change` | 2004 / 3002 / 3001 |
| `AADRiskyUsers` | `azure_aad_risky_users` | `azure_aad_risky_users` | `detection_finding` + `account_change` | 2004 / 3001 |
| `AzureActivity` | `azure_activity_logs` | `azure_activity_logs` | `api_activity` (primary), some rows route to `detection_finding` | 6003 / 2004 |
| `AzureDiagnostics` / `AzureNetworkAnalytics_CL` (Azure Firewall — NetworkRule) | `azure_firewall` | `azure_firewall` | `network_activity` | 4001 |
| `AzureDiagnostics` (Azure Firewall — DNS proxy) | `azure_firewall` | `azure_firewall` | `dns_activity` | 4003 |
| `AzureDiagnostics` (Azure Firewall — ThreatIntel) | `azure_firewall` | `azure_firewall` | `detection_finding` | 2004 |
| VNet Flow Logs (`AzureNetworkAnalytics_CL`, NSG flow logs v2) | `azure_vnet_flow_logs` | `azure_vnet_flow_logs` | `network_activity` | 4001 |
| `SecurityEvent` (Windows Event Log) | `windows_events_xml` | per-source silver | drives off `EventID` — see Windows table below | varies |
| `WindowsEvent` (XML-based) | `windows_events_xml` | same | same | varies |
| `SecurityAlert` (Sentinel-managed alerts) | none directly — Defender alerts come via `microsoft_defender_xdr`'s `AlertInfo`/`AlertEvidence` | `mde_alert_events` | `detection_finding` | 2004 |
| `SecurityIncident` | not currently ingested — check with the user | — | — | — |
| `OfficeActivity` (M365 audit) | not currently ingested — confirm with the user | — | — | — |
| `ThreatIntelligenceIndicator` | typically loaded as a dataset/enrichment table | — | — | — |
| `Heartbeat`, `Usage`, `Operation` (Log Analytics platform) | **out of scope** — use Databricks `system.*` tables for Lakewatch platform health | — | — | — |

## How to filter to a specific Sentinel source on gold

Gold tables hold events from multiple products. A source filter is essential. **Prefer
`metadata.log_name`** (the preset's `sourceType`) — it's a short, indexed-friendly string that
gives the best Delta data-skipping. Use `metadata.log_provider` (the preset's `source`) as a
coarser cut, and `metadata.product.name` only when you need to discriminate inside a `log_name`
family.

For Entra specifically, `metadata.product.name` is also unreliable as a filter — the preset sets
it dynamically from the source `category` (e.g., `'SigninLogs'`, `'NonInteractiveUserSignInLogs'`,
`'Microsoft Entra ID'`, `'Azure AD B2C'`). `log_name` is the only stable Entra filter.

| To isolate | Filter |
| --- | --- |
| All Entra sign-ins on `authentication` | `metadata.log_name = 'microsoft_entra_authentication'` |
| Only interactive sign-ins | add `metadata.event_code = 'SigninLogs'` (or equivalent category) |
| Only non-interactive | `metadata.event_code = 'NonInteractiveUserSignInLogs'` |
| Only service principal | `metadata.event_code = 'AADServicePrincipalSignInLogs'` |
| Entra audit (account changes) | `metadata.log_name = 'microsoft_entra_account_change'` |
| Entra group changes | `metadata.log_name = 'microsoft_entra_group_management'` |
| Entra app/SP changes | `metadata.log_name = 'microsoft_entra_entity_management'` |
| Entra role assignments | `metadata.log_name = 'microsoft_entra_user_access'` |
| Microsoft Graph activity | `metadata.log_name = 'microsoft_entra_api_activity'` |
| Azure Activity (ARM control plane) | `metadata.log_name = 'azure_activity_logs'` |
| Azure Firewall network rules | `metadata.log_name = 'azure_firewall'` and the relevant category |
| Azure VNet Flow Logs | `metadata.log_name = 'azure_vnet_flow_logs'` |
| All Defender XDR events (Endpoint + Identity + O365 + alerts) | `metadata.log_name = 'defender_xdr'` |
| Defender for Endpoint events specifically | `metadata.log_name = 'defender_xdr' AND metadata.product.name = 'Microsoft Defender for Endpoint'` — needed only when the gold table mixes Endpoint with another Defender product (e.g., `authentication` also has `IdentityLogonEvents`) |
| All Microsoft sources, vendor-wide | `metadata.log_provider = 'microsoft'` |

When in doubt, run `SELECT DISTINCT metadata.log_name, metadata.log_provider,
metadata.product.name, metadata.event_code FROM <catalog>.gold.<table> WHERE time >=
current_timestamp() - INTERVAL 1 DAY` against the user's workspace and pick the filter that
matches the rows you want.

## Working directly against silver (often the simplest path)

Because Entra's gold mapping is complex (one Sentinel `AuditLogs` row routes to four different
gold tables based on `Category`), it is often **simpler** to translate a Sentinel `AuditLogs` or
`SigninLogs` query against the matching `microsoft_entra_*` silver table — column names there
already match the Azure Monitor schema (`operationName`, `callerIpAddress`, `resultType`,
`identity`, `category`, etc.), so the translation is close to a renaming exercise.

Use silver when:

- The query reads columns that don't have clean OCSF mappings (e.g., `OperationVersion`,
  `CorrelationId`).
- The query relies on `Category` discrimination that splits across multiple gold tables.
- The user prefers a 1:1 column-level translation and doesn't need cross-vendor normalization.

Use gold when:

- The query is meant to run across multiple vendors (e.g., "all failed logins" — should pick up
  both Entra and Defender).
- The query is feeding a detection rule that ought to be vendor-agnostic.

Whenever you choose silver over gold, **say so explicitly in the mapping notes** so the reviewer
knows why.

## Column mappings — `SigninLogs` → `authentication` (gold)

| Sentinel `SigninLogs` | Gold `authentication` |
| --- | --- |
| `TimeGenerated` | `time` |
| `UserPrincipalName` | `user.name` |
| `UserId` | `user.uid` |
| `UserType` | `user.type` |
| `AppId` | `actor.app_uid` |
| `AppDisplayName` | `actor.app_name` |
| `IPAddress` | `src_endpoint.ip` |
| `Location` (country code) | `src_endpoint.location.country` |
| `LocationDetails.city` | `src_endpoint.location.city` |
| `DeviceDetail.deviceId` | `device.uid` (if mapped — verify with `DESCRIBE`) |
| `ResultType` (`0` = success, anything else = failure) | drives `status_id` (1 = Success, 2 = Failure) and `status` |
| `ResultDescription` | `status_detail` |
| `ConditionalAccessStatus` | inside `metadata.event_code` or `unmapped` |
| `RiskState`, `RiskLevelAggregated` | inside `unmapped` (no first-class OCSF mapping) |

## Column mappings — `SigninLogs` → `microsoft_entra_authentication` (silver, simpler path)

Silver keeps the Azure Monitor / Entra raw schema. Reach raw fields with
`try_variant_get(data, '$.<field>', '<type>')` if needed, but the preset already projects the
common ones as top-level columns:

| Sentinel `SigninLogs` | Silver `microsoft_entra_authentication` |
| --- | --- |
| `TimeGenerated` | `time` |
| `OperationName` | `operationName` |
| `Category` | `category` |
| `ResultType` | `resultType` |
| `ResultDescription` | `resultDescription` |
| `ResultSignature` | `resultSignature` |
| `CallerIpAddress` | `callerIpAddress` |
| `Identity` | `identity` |
| `Location` | `location` |
| `CorrelationId` | `correlationId` |
| `TenantId` | `tenantId` |
| (anything else) | `try_variant_get(data, '$.properties.<FieldName>', '<type>')` |

## Column mappings — `AuditLogs` → multiple gold tables

`AuditLogs` is heterogeneous. Lakewatch routes by `Category` (and sometimes `OperationName`) to
different silver / gold tables. The cleanest translation is usually to **filter on
`metadata.log_name`** rather than try to query a single table.

| AuditLogs `Category` | Gold table | Silver table |
| --- | --- | --- |
| `UserManagement` | `account_change` (3001) | `microsoft_entra_account_change` |
| `GroupManagement` | `group_management` (3006) | `microsoft_entra_group_management` |
| `ApplicationManagement`, `ServicePrincipalManagement`, similar | `entity_management` (3004) | `microsoft_entra_entity_management` |
| `RoleManagement`, directory role assignments | `user_access` (3005) | `microsoft_entra_user_access` |
| `MicrosoftGraphActivity` | `api_activity` (6003) | `microsoft_entra_api_activity` |

Common columns across these:

| Sentinel `AuditLogs` | Gold path |
| --- | --- |
| `TimeGenerated` | `time` |
| `OperationName` | `activity_name` / informs `activity_id`, also `metadata.event_code` |
| `InitiatedBy.user.userPrincipalName` | `actor.user.name` |
| `InitiatedBy.app.appId` | `actor.app_uid` |
| `TargetResources[0].userPrincipalName` | `user.name` (account_change), `entity.user.name` (entity_management) |
| `TargetResources[0].displayName` | `group.name` (group_management), `entity.name` (entity_management) |
| `TargetResources[0].id` | `user.uid` / `group.uid` / `entity.uid` |
| `Result` (`success` / `failure`) | `status_id` (1 / 2), `status` |
| `ModifiedProperties` | inside `unmapped` (array of `{name, oldValue, newValue}`) |
| `CorrelationId` | `metadata.correlation_uid` |

If the source KQL keys off `Category`, treat that as a strong hint about which gold table to
target — and use `metadata.log_name` to filter on the Lakewatch side.

## Column mappings — `AzureActivity` → `api_activity`

Fed by the `azure_activity_logs` preset.

| Sentinel `AzureActivity` | Gold `api_activity` |
| --- | --- |
| `TimeGenerated` | `time` |
| `Caller` | `actor.user.name` |
| `CallerIpAddress` | `src_endpoint.ip` |
| `OperationName` / `OperationNameValue` | `api.operation` |
| `ResourceId` | `api.target.uid` (or `resources[0].uid`) |
| `ResourceGroup` | `api.target.group.name` |
| `SubscriptionId` | `cloud.account.uid` |
| `Category` | `api.service.name` (e.g., `Administrative`, `Policy`) |
| `ActivityStatusValue` (`Success`/`Failed`) | `status_id` / `status` |
| `Properties` (JSON blob) | inside `unmapped` |

Filter: `metadata.log_name = 'azure_activity_logs'`.

## Column mappings — Azure Firewall / VNet Flow

Both feed `network_activity` (gold class 4001). Azure Firewall additionally feeds `dns_activity`
(4003) for DNS proxy rows, and `detection_finding` (2004) for ThreatIntel rule hits.

| Source field | Gold `network_activity` |
| --- | --- |
| `TimeGenerated` | `time` |
| Source IP | `src_endpoint.ip` |
| Source port | `src_endpoint.port` |
| Destination IP | `dst_endpoint.ip` |
| Destination port | `dst_endpoint.port` |
| Protocol | `connection_info.protocol_name` |
| Action (Allow / Deny) | `disposition` / `disposition_id` |
| Bytes sent / received | `traffic.bytes_in` / `traffic.bytes_out` |

Filter: `metadata.log_name = 'azure_firewall'` or `metadata.log_name = 'azure_vnet_flow_logs'`.

## Column mappings — `AADUserRiskEvents` / `AADRiskyUsers`

Fed by the `azure_aad_user_risk_events` and `azure_aad_risky_users` presets. These have multiple
gold destinations:

- `detection_finding` (2004) — the risk event itself as a finding
- `authentication` (3002) — the underlying sign-in row (only `AADUserRiskEvents`)
- `account_change` (3001) — the user-state change

Pick the gold table that matches the intent of the KQL query. If the query is reading risk fields
(`RiskLevel`, `RiskState`, `RiskDetail`), `detection_finding` is usually the right destination. If
the query joins risk events to sign-in details, you may need both `detection_finding` and
`authentication`.

| Sentinel field | Gold path (in `detection_finding`) |
| --- | --- |
| `TimeGenerated` | `time` |
| `UserPrincipalName` | `evidences[].user.name` or in `unmapped` |
| `RiskLevel` (Low/Medium/High) | drives `severity_id` / `severity` |
| `RiskState` (atRisk, confirmedCompromised, …) | `finding_info.types` / `status` |
| `RiskDetail` | `finding_info.desc` |
| `Source` | `metadata.product.feature.name` |

Filter: `metadata.log_name = 'azure_aad_user_risk_events'` (or `azure_aad_risky_users`).

## Column mappings — `SecurityEvent` (Windows Event Log) → varies

Windows Event Log events from Sentinel go through Lakewatch's `windows_events_xml` preset. There's
no single gold mapping; the right destination depends on the EventID.

Common patterns:

| EventID | KQL intent | Lakewatch destination |
| --- | --- | --- |
| 4624, 4625 | logons | `authentication` (gold) |
| 4688 | process creation | `process_activity` (gold) |
| 4720, 4722, 4723, 4724, 4725, 4726, 4738 | account changes | `account_change` (gold) |
| 4727, 4728, 4729, 4730, 4731, 4732, 4733, 4734, 4735 | group management | `group_management` (gold) |
| 5140, 5145 | file share access | `file_activity` (gold) |
| other | falls back to silver — query the `windows_events_xml` silver |  |

`EventID` lives in the silver table directly; on gold it becomes part of `metadata.event_code`.

## Column mappings — `SecurityAlert` → `detection_finding`

Sentinel's `SecurityAlert` table is **not directly ingested** by a Lakewatch preset today. The
closest equivalent is Defender's `AlertInfo` + `AlertEvidence`, ingested by the
`microsoft_defender_xdr` preset and normalized to `detection_finding`. See
`defender-table-mapping.md` for the Defender alert mapping.

If the user's Sentinel query reads `SecurityAlert` rows that come from a non-Defender source
(e.g., MCAS, Azure Defender for Cloud), stop and ask — that data may not be in Lakewatch.

## When Sentinel uses functions or watchlists

KQL inside Sentinel often calls workspace-specific functions (`SecurityEvent | invoke
SecurityEvent_normalized()`) or watchlists (`_GetWatchlist("HighValueAssets")`). These don't have
direct Lakewatch equivalents:

- **Workspace functions**: ask the user to share the function definition, then inline it into the
  translation.
- **Watchlists**: in Lakewatch, this is usually a dataset (delta table) imported via a content
  pack, or a Unity Catalog table. Ask the user which table holds the equivalent data and rewrite
  the call as a SQL join.

Don't silently drop these — the resulting SQL will produce different results from the KQL and the
user will lose trust.

## Catalog and schema

`<catalog>.gold.<table>` and `<catalog>.silver.<table>`. The catalog name **varies by
environment** (e.g., `lw-prod-catalog`, `lakewatch_<env>`, sandbox names) — confirm with the user
or `SHOW CATALOGS`. Many Lakewatch SQL contexts default to `gold` as the active schema, so the
catalog/schema prefix can often be elided in shipped queries — but include it when in doubt.
