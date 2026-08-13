# Sumo Logic `_sourceCategory` → Lakewatch presets / tables

This file maps common **Sumo Logic Source Categories** (and the data behind them) to the Lakewatch
presets that ingest the same data, and to the OCSF gold tables they produce.

> **`_sourceCategory` is customer-defined.** Unlike Splunk sourcetypes (which are vendor-fixed
> strings like `aws:cloudtrail`), Sumo Source Categories are **free-form labels chosen at
> collection time** by each customer — `prod/aws/cloudtrail`, `AWS/CloudTrail/Prod`,
> `security-cloudtrail`, etc. The strings in the left column below are *typical conventions*, not
> guaranteed values. **Identify the data source by what the logs actually are**, not by pattern-
> matching the category string. When unsure, ask the user which data source a `_sourceCategory`
> represents.

> **Verify before relying.** The column mappings are extracted from the content-marketplace
> presets. Different presets promote different source fields to typed OCSF paths vs keep them in
> `unmapped` (VARIANT). When in doubt, `DESCRIBE EXTENDED <catalog>.gold.<table>` or read the
> preset YAML directly.

## Data source → preset → gold table index

| Data behind the `_sourceCategory` (typical category text) | Lakewatch preset (`metadata.log_name`) | Silver table(s) | Gold tables produced |
| --- | --- | --- | --- |
| AWS CloudTrail (`*/cloudtrail`, `aws/cloudtrail`) | `cloudtrail` | `aws_cloudtrail` | `api_activity` (primary), `authentication` (ConsoleLogin), `account_change` (IAM users), `group_management` (IAM groups), `entity_management` (resource lifecycle), `datastore_activity` (S3/DynamoDB/RDS), `file_hosting_activity` (S3 object ops), `network_activity` (VPC/EC2 control plane) |
| AWS VPC Flow (`*/vpcflow`, `aws/vpc`) | `aws_vpc_flowlogs_v2` | `aws_vpc_flowlogs_v2` | `network_activity` |
| AWS GuardDuty (`*/guardduty`) | `aws_guardduty` | `aws_guardduty` | `detection_finding` |
| AWS WAF (`*/waf`) | `aws_waf_v2` | `aws_waf_v2` | `http_activity`, `detection_finding` |
| Windows Security events (`*/windows/security`, `OS/Windows/Security`) | `windows_events_xml` | `windows_events_xml` | `authentication`, `process_activity`, `account_change`, `group_management`, `file_activity` (per EventID) |
| CrowdStrike Falcon (`*/crowdstrike`, `edr/crowdstrike`) | `falcon_per_event_schema` | `crowdstrike`, `crowdstrike_process_rollup` | `process_activity`, `file_activity`, `network_activity`, `dns_activity`, `authentication`, `account_change`, `group_management`, `script_activity`, `detection_finding` |
| Palo Alto traffic (`*/palo*/traffic`, `firewall/palo`) | `firewall` (paloalto) | `paloalto_traffic` | `network_activity` |
| Palo Alto threat (`*/palo*/threat`) | `firewall` (paloalto) | `paloalto_threat` | `detection_finding` |
| Cisco ASA (`*/cisco/asa`, `network/asa`) | `asa` | `cisco_asa` | `network_activity`, `authentication`, `http_activity`, `detection_finding` |
| Zeek/Bro conn (`*/zeek/conn`, `bro/conn`) | `zeek_conn` | `zeek_conn` | `network_activity` |
| Zeek/Bro dns | `zeek_dns` | `zeek_dns` | `dns_activity` |
| Zeek/Bro http | `zeek_http` | `zeek_http` | `http_activity` |
| Check Point firewall (`*/checkpoint`) | `checkpoint_firewall_syslog` | `checkpoint_firewall_syslog` | `network_activity`, `detection_finding` |
| Cloudflare HTTP (`*/cloudflare/http`) | `cloudflare_httpreq` | `cloudflare_httpreq` | `http_activity` |
| Cloudflare firewall | `cloudflare_firewall_events` | `cloudflare_firewall_events` | `network_activity`, `detection_finding` |
| Zscaler ZPA (`*/zscaler/zpa`) | `zscaler_zpa` | `zscaler_zpa` | varies |
| Zscaler ZIA / web (`*/zscaler/web`, `zscalernss`) | not directly — Lakewatch has ZPA (Private Access), not ZIA. Check with the user. | — | — |
| Apache access (`*/apache/access`) | `apache_access` (verify path) | varies | `http_activity` |
| GCP audit (`*/gcp/audit`) | `gcp` (verify path) | varies | `api_activity` |
| GitHub audit (`*/github/audit`) | `github` (verify path) | varies | `api_activity`, `account_change`, `group_management` |
| Sumo audit (`_index=sumologic_audit_events`) | not a security-log source — Sumo platform audit. No Lakewatch preset; flag. | — | — |

For sources not in the table, check `packs/<vendor>/presets/<preset>/preset.yaml` in the
antimatter content-marketplace repo (or `DESCRIBE` the candidate tables) before guessing.

## Sumo metadata fields → Lakewatch

Sumo automatic metadata fields don't have direct OCSF homes:

| Sumo field | Lakewatch path | Notes |
| --- | --- | --- |
| `_messageTime` | `time` | canonical event timestamp |
| `_receiptTime` | `metadata.logged_time` / `metadata.processed_time` | ingest time |
| `_raw` | `raw_data` (VARIANT) on some gold tables; `data` (VARIANT) on silver | not preserved by every preset |
| `_sourceCategory` | **no column** — map to `metadata.log_name` (the preset) | this is the key mental switch: filter on the preset, not on a category string |
| `_source` | `metadata.log_name` (closest) | Sumo Source configuration name |
| `_sourceName` | `metadata.log_provider` or (varies) | often a file path / input name in Sumo |
| `_sourceHost` | `device.hostname` (endpoint events) / `src_endpoint.hostname` (network) / `dst_endpoint.hostname` (auth targeting a host) | role-dependent |
| `_collector` | no equivalent | Sumo Collector name (agent/platform field) |
| `_index` / `_view` | no equivalent | Sumo Partition / Scheduled View — storage/acceleration; ask which Lakewatch table holds the data |
| `_messageId` / `_blockId` | no equivalent | Sumo internal IDs |
| `_size` | no equivalent | message byte size |

## Column mappings — CloudTrail → `api_activity` (primary)

The `cloudtrail` preset promotes the CloudTrail JSON fields you would otherwise `json`-extract in
Sumo. **Delete the `json` extraction steps** and use these paths:

| CloudTrail JSON (Sumo `json "…"`) | Gold `api_activity` | Notes |
| --- | --- | --- |
| `eventTime` / `_messageTime` | `time` | |
| `eventSource` | `api.service.name` | e.g. `s3.amazonaws.com` |
| `eventName` | `api.operation`, `metadata.event_code` | |
| `awsRegion` | `cloud.region` | |
| `recipientAccountId` | `cloud.account.uid` | |
| `userIdentity.userName` | `actor.user.name` | |
| `userIdentity.arn` | `actor.user.uid` | |
| `userIdentity.accountId` | `actor.user.account.uid` | |
| `userIdentity.type` | `actor.user.type` | |
| `sourceIPAddress` | `src_endpoint.ip` | |
| `userAgent` | `http_request.user_agent` | |
| `errorCode` / `errorMessage` | `status_code` / `status_detail`; success/fail → `status_id` (1/2) | |
| `requestParameters` | `api.request.data` (VARIANT) | full payload preserved |
| `responseElements` | `api.response.data` (VARIANT) | |
| `eventID` | `metadata.uid` | |
| `resources[]` | `resources` (array of structs) | |

**Routing**: ConsoleLogin → `authentication`; IAM `Create*`/`Delete*` → `account_change` /
`group_management`; S3/DynamoDB/RDS data events → `datastore_activity` / `file_hosting_activity`.
Use `metadata.log_name = 'cloudtrail'` plus the gold table to scope.

## Column mappings — Windows Security → `authentication` (EventID 4624/4625)

| Windows field (Sumo `parse`/`keyvalue`) | Gold `authentication` | Notes |
| --- | --- | --- |
| `_messageTime` | `time` | |
| `EventCode` / `EventID` | `metadata.event_code` | filter `'4624'` (success) / `'4625'` (failure) |
| `TargetUserName` / `Account_Name` | `user.name` | |
| `TargetDomainName` / `Account_Domain` | `user.account.name` | |
| `TargetUserSid` | `user.uid` | |
| `IpAddress` / `Source_Network_Address` | `src_endpoint.ip` | |
| `WorkstationName` | `src_endpoint.hostname` | |
| `host` (target machine) | `dst_endpoint.hostname` | |
| `LogonType` | `logon_type_id` / `logon_type` | |
| Success/Failure (from EventID) | `status_id` (1/2), `status` | derived |

### Windows EventID routing

| EventID | Gold table | Filter |
| --- | --- | --- |
| 4624, 4625, 4634, 4647, 4648 | `authentication` | `metadata.event_code IN (…)` |
| 4688, 4689 | `process_activity` | |
| 4720, 4722–4726, 4738 | `account_change` | |
| 4727–4735, 4737, 4754–4758, 4764 | `group_management` | |
| 5140, 5145 | `file_activity` | |

## Column mappings — Palo Alto traffic → `network_activity`

| Palo field (Sumo `csv`/`parse`) | Gold `network_activity` | Notes |
| --- | --- | --- |
| `_messageTime` | `time` | |
| `src` / `src_ip` | `src_endpoint.ip` | |
| `src_port` | `src_endpoint.port` | |
| `dst` / `dest_ip` | `dst_endpoint.ip` | |
| `dst_port` / `dest_port` | `dst_endpoint.port` | |
| `protocol` | `connection_info.protocol_name` | |
| `action` (allow/deny/drop) | `disposition`, `disposition_id`, `action`, `action_id` | |
| `bytes_sent`/`bytes_received` | `traffic.bytes_out`/`traffic.bytes_in` | |
| `user` | `user.name` (User-ID mapped) | |
| `app` | `app_name` (where promoted) | |
| `rule` | `policy.name` (where promoted) | |

Use `metadata.log_name = 'firewall'` plus `metadata.product.vendor_name = 'Palo Alto Networks'`
to distinguish Palo from other firewalls if multiple are ingested.

## Column mappings — CrowdStrike → various gold tables

CrowdStrike FDR records route by event name. Use `metadata.log_name = 'falcon_per_event_schema'`
plus `metadata.event_code` (the CS event name) to discriminate.

| CS event family | Gold table | Common CS field → OCSF path |
| --- | --- | --- |
| `ProcessRollup2` | `process_activity` | `CommandLine`→`process.cmd_line`; `FileName`→`process.name`; `ImageFileName`→`process.file.path`; `ParentBaseFileName`→`actor.process.name`; `UserName`→`process.user.name`; `aid`→`device.uid`; `ComputerName`→`device.hostname`; `SHA256HashData`→`process.file.hashes` |
| `DnsRequest` | `dns_activity` | `DomainName`→`query.hostname`; `RequestType`→`query.type` |
| `NetworkConnect*` | `network_activity` | `LocalAddressIP4`→`src_endpoint.ip`; `RemoteAddressIP4`→`dst_endpoint.ip` |
| `UserLogon*` | `authentication` | `UserName`→`user.name`; `RemoteAddressIP4`→`src_endpoint.ip` |
| `FileWritten`/`FileDeleted` | `file_activity` | `TargetFileName`→`file.path` |
| `Detection*` | `detection_finding` | severity, tactic, technique fields |
