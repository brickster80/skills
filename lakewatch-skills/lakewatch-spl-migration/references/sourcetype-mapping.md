# Splunk sourcetypes → Lakewatch presets / tables

This file maps the most common **Splunk sourcetypes** (and related `index=`/`source=` filters) to
the Lakewatch presets that ingest the same data, and to the OCSF gold tables they produce.

> **Verify before relying.** The column mappings below are extracted from the content-marketplace
> presets. Different Lakewatch presets choose differently which source fields to promote to typed
> OCSF struct paths and which to keep in `unmapped` (a VARIANT column). When in doubt, `DESCRIBE
> EXTENDED <catalog>.gold.<table>` or read the preset YAML directly.

## Sourcetype → preset → gold table index

| Splunk sourcetype / source | Lakewatch preset (`metadata.log_name`) | Silver table(s) | Gold tables produced |
| --- | --- | --- | --- |
| `WinEventLog:Security`, `WinEventLog:System`, `WinEventLog:Application` (legacy MMA) | `windows_events_xml` | `windows_events_xml` | `authentication`, `process_activity`, `account_change`, `group_management`, `file_activity` (per EventID) |
| `XmlWinEventLog:*` (AMA) | `windows_events_xml` | same | same |
| `crowdstrike:falconhost:json` (FDR / Event Stream) | `falcon_per_event_schema` | `crowdstrike`, `crowdstrike_process_rollup` | `process_activity`, `file_activity`, `network_activity`, `dns_activity`, `authentication`, `account_change`, `group_management`, `scheduled_job_activity`, `script_activity`, `detection_finding` |
| `aws:cloudtrail` | `cloudtrail` | `aws_cloudtrail` | `api_activity` (primary), `authentication` (ConsoleLogin), `account_change` (IAM users), `group_management` (IAM groups), `entity_management` (resource lifecycle), `datastore_activity` (S3 / DynamoDB / RDS), `file_hosting_activity` (S3 object ops), `network_activity` (VPC/EC2 control plane) |
| `aws:vpcflow` | `aws_vpc_flowlogs_v2` | `aws_vpc_flowlogs_v2` | `network_activity` |
| `aws:waf` | `aws_waf_v2` | `aws_waf_v2` | `http_activity`, `detection_finding` |
| `aws:guardduty` | `aws_guardduty` | `aws_guardduty` | `detection_finding` |
| `aws:network:firewall` | `aws_network_firewall` | `aws_network_firewall` | `network_activity`, `detection_finding` |
| `pan:traffic` | `firewall` (paloalto) | `paloalto_traffic` | `network_activity` |
| `pan:threat` | `firewall` (paloalto) | `paloalto_threat` | `detection_finding` |
| `pan:userid` | `firewall` (paloalto) | `paloalto_userid` | `authentication` |
| `pan:hipmatch` | `firewall` (paloalto) | `paloalto_hipmatch` | `detection_finding` |
| `pan:system`, `pan:config` | not currently ingested as a dedicated preset — check with the user | — | — |
| `cisco:asa` | `asa` | `cisco_asa` | `network_activity`, `authentication`, `process_activity`, `detection_finding`, `http_activity` |
| `cisco:firepower`, `cisco:ftd:syslog` | `cisco_firepower` | `cisco_firepower` | `network_activity`, `authentication`, `detection_finding` (varies by message) |
| `cisco:ios` | `cisco_ios` | `cisco_ios` | varies (basic syslog routing) |
| `cisco:umbrella` | `cisco_umbrella_dns` | `cisco_umbrella_dns` | `dns_activity` |
| `bro:conn:json`, `zeek:conn:json` | `zeek_conn` | `zeek_conn` | `network_activity` |
| `bro:dns:json`, `zeek:dns:json` | `zeek_dns` | `zeek_dns` | `dns_activity` |
| `bro:http:json`, `zeek:http:json` | `zeek_http` | `zeek_http` | `http_activity` |
| `bro:dhcp:json` | `zeek_dhcp` | `zeek_dhcp` | `dhcp_activity` |
| `bro:ssh:json` | `zeek_ssh` | `zeek_ssh` | `ssh_activity` |
| `bro:smtp:json` | `zeek_smtp` | `zeek_smtp` | `email_activity` |
| `bro:ntlm:json`, `bro:kerberos:json` | `zeek_ntlm` / `zeek_kerberos` | same | `authentication` |
| `bro:tunnel:json` | `zeek_tunnel` | `zeek_tunnel` | (silver-only; no gold yet — confirm) |
| `bro:websocket:json` | `zeek_websocket` | `zeek_websocket` | (silver-only) |
| `bro:quic:json` | `zeek_quic` | `zeek_quic` | (silver-only) |
| `bro:socks:json` | `zeek_socks` | `zeek_socks` | (silver-only) |
| `cp:firewall:syslog` (Check Point) | `checkpoint_firewall_syslog` | `checkpoint_firewall_syslog` | `network_activity`, `detection_finding` |
| `zscalernss:web`, `zscaler:zia:*` | not directly — Lakewatch has `zscaler_zpa` (Private Access), not ZIA. Check with the user. | — | — |
| `zscaler:zpa:*` | `zscaler_zpa` | `zscaler_zpa` | varies |
| `cloudflare:httpreq` / `cloudflare:logpush` | `cloudflare_httpreq` | `cloudflare_httpreq` | `http_activity` |
| `cloudflare:firewall` | `cloudflare_firewall_events` | `cloudflare_firewall_events` | `network_activity`, `detection_finding` |
| `cloudflare:gateway_dns` | `cloudflare_gateway_dns` | `cloudflare_gateway_dns` | `dns_activity` |
| `cb:edr:filemod` (Carbon Black EDR) | `carbonblack_edr_filemod` | `carbonblack_edr_filemod` | `file_activity` |
| `cb:edr:netconn` | `carbonblack_edr_netconn` | `carbonblack_edr_netconn` | `network_activity` |
| `apache:access_combined`, `apache:access_combined_wcookie` | `apache_access` (verify path) | varies | `http_activity` |
| `squid:access` | `squid_access_log` | `squid_access_log` | `http_activity` |
| `github:enterprise:audit` | `github` (verify path) | varies | `api_activity`, `account_change`, `group_management` |
| `atlassian:jira:*` / `atlassian:confluence:*` | `atlassian` (verify path) | varies | varies |
| `gcp:audit:*` | `gcp` (verify path) | varies | `api_activity` |

For sourcetypes not in the table, check `packs/<vendor>/presets/<preset>/preset.yaml` in the
antimatter content-marketplace repo (or `DESCRIBE` the candidate tables) before guessing.

## Splunk-only universal fields

Splunk universal-forwarder fields don't have direct OCSF homes:

| Splunk field | Lakewatch path | Notes |
| --- | --- | --- |
| `_time` | `time` | canonical timestamp |
| `_raw` | `raw_data` (VARIANT) on some gold tables; `data` (VARIANT) on silver | Defender preset preserves `raw_data` on `process_activity`/`file_activity`/etc.; others don't |
| `host` | varies — `device.hostname` (endpoint events), `src_endpoint.hostname` (network events), `dst_endpoint.hostname` (auth events targeting a host) | the right OCSF path depends on the role of the host in the event |
| `source` | `metadata.log_provider` (vendor-level) or `metadata.log_name` (preset-level) | Splunk's `source` is usually a file path / input name; Lakewatch's `log_*` are higher-level |
| `sourcetype` | `metadata.log_name` | this is the closest analog — both identify "what kind of event is this" |
| `index` | no direct equivalent | Splunk's `index` is a storage partition; Lakewatch uses Unity Catalog schemas (`gold`, `silver`) plus the `metadata.log_*` filters |
| `splunk_server` | no equivalent | platform field |

## Column mappings — `WinEventLog:Security` → `authentication` (EventCode 4624/4625)

Logon events from the `windows_events_xml` preset. The preset reads Windows Event XML and
populates OCSF struct paths.

| Splunk (`WinEventLog:Security`) | Gold `authentication` | Notes |
| --- | --- | --- |
| `_time` | `time` | |
| `EventCode` | `metadata.event_code` | filter on `'4624'` (success) / `'4625'` (failure) |
| `Account_Name` / `TargetUserName` | `user.name` | |
| `Account_Domain` / `TargetDomainName` | `user.account.name` | |
| `Security_ID` / `TargetUserSid` | `user.uid` | |
| `Source_Network_Address` / `IpAddress` | `src_endpoint.ip` | |
| `Source_Port` | `src_endpoint.port` | |
| `Workstation_Name` / `WorkstationName` | `src_endpoint.hostname` | |
| `host` (target machine) | `dst_endpoint.hostname` | the host that received the logon |
| `Logon_Type` / `LogonType` | `logon_type_id` / `logon_type` | also drives `auth_protocol` per Windows logon type |
| Success / Failure (from `EventCode`) | `status_id` (1 / 2), `status` | derived |
| `Failure_Reason` (4625 only) | `status_detail` | |
| `Sub_Status` (4625 only) | `status_code` | the integer NTSTATUS code |

For non-4624/4625 events (e.g., 4720 account create, 4732 group member add) the preset routes
to a different gold table — see the per-EventID table below.

## Windows EventID routing

| EventID | Lakewatch gold table | Filter |
| --- | --- | --- |
| 4624 (logon success), 4625 (logon failure) | `authentication` | `metadata.event_code IN ('4624', '4625')` |
| 4634, 4647, 4648 (logoff / other-creds logon) | `authentication` | corresponding event codes |
| 4688 (process create), 4689 (process exit) | `process_activity` | `metadata.event_code IN ('4688', '4689')` |
| 4720, 4722, 4723, 4724, 4725, 4726, 4738 (user mgmt) | `account_change` | |
| 4727, 4728, 4729, 4730, 4731, 4732, 4733, 4734, 4735, 4737, 4754, 4755, 4756, 4757, 4758, 4764 (group mgmt) | `group_management` | |
| 5140, 5145 (file-share access) | `file_activity` | |
| 1102 (audit log cleared) | `detection_finding` (sometimes) — confirm with the preset YAML | |
| other | check the silver `windows_events_xml` table |  |

## Column mappings — `crowdstrike:falconhost:json` → various gold tables

CrowdStrike FDR (Falcon Data Replicator) JSON ships as per-event-schema records. The
`falcon_per_event_schema` preset routes them by event name into different OCSF gold classes.

| Splunk FDR event family | Gold table | Common CS field → OCSF path |
| --- | --- | --- |
| `ProcessRollup2` | `process_activity` | `CommandLine` → `process.cmd_line`; `FileName` → `process.name`; `ImageFileName` → `process.file.path`; `ParentBaseFileName` → `actor.process.name`; `UserName` → `process.user.name`; `aid` → `device.uid`; `ComputerName` → `device.hostname`; `SHA256HashData` → `process.file.hashes` (as SHA-256 entry) |
| `DnsRequest` | `dns_activity` | `DomainName` → `query.hostname`; `RequestType` → `query.type` |
| `NetworkConnect*` | `network_activity` | `LocalAddressIP4` → `src_endpoint.ip`; `RemoteAddressIP4` → `dst_endpoint.ip`; `LocalPort`/`RemotePort` |
| `UserLogon*` | `authentication` | `UserName` → `user.name`; `RemoteAddressIP4` → `src_endpoint.ip` |
| `FileWritten`, `FileDeleted`, `FileRenamed` | `file_activity` | `TargetFileName` → `file.path` |
| `ScriptControlScanInfo`, `Script*` | `script_activity` | `ScriptContent` → `script_content` (where promoted) |
| `Detection*` | `detection_finding` | severity, tactic, technique fields |

Use `metadata.log_name = 'falcon_per_event_schema'` as the source filter and add filters on
`metadata.event_code` (which carries the CrowdStrike event name) to discriminate within the
preset.

## Column mappings — `aws:cloudtrail` → `api_activity` (primary)

| Splunk (`aws:cloudtrail`) | Gold `api_activity` | Notes |
| --- | --- | --- |
| `_time` / `eventTime` | `time` | |
| `eventSource` | `api.service.name` | e.g., `s3.amazonaws.com` |
| `eventName` | `api.operation`, `metadata.event_code` | |
| `awsRegion` | `cloud.region` | |
| `recipientAccountId` | `cloud.account.uid` | |
| `userIdentity.userName` | `actor.user.name` | |
| `userIdentity.arn` | `actor.user.uid` | (sometimes; the ARN is a longer identifier than the user uid) |
| `userIdentity.accountId` | `actor.user.account.uid` | |
| `userIdentity.type` | `actor.user.type` | |
| `sourceIPAddress` | `src_endpoint.ip` | |
| `userAgent` | `http_request.user_agent` | when present |
| `errorCode` / `errorMessage` | `status_code` / `status_detail` | success vs failure → `status_id` (1/2) |
| `requestParameters` | `api.request.data` (VARIANT) | nested JSON, full payload preserved |
| `responseElements` | `api.response.data` (VARIANT) | |
| `eventID` | `metadata.uid` | |
| `eventCategory` | `api.service.category` (where promoted) | e.g., `Management`, `Data` |
| `eventType` | `metadata.event_code` (when more specific than `eventName`) | |
| `resources[]` | `resources` (array of structs) | each resource has `uid`, `type`, `account_uid` |

**Routing**: ConsoleLogin events route to `authentication`; IAM `Create*`/`Delete*` events route
to `account_change` or `group_management` depending on the entity; S3 / DynamoDB / RDS data
events route to `datastore_activity` or `file_hosting_activity`. Use
`metadata.log_name = 'cloudtrail'` plus the gold table to scope.

## Column mappings — `pan:traffic` (Palo Alto) → `network_activity`

| Splunk (`pan:traffic`) | Gold `network_activity` | Notes |
| --- | --- | --- |
| `_time` | `time` | |
| `src` / `src_ip` | `src_endpoint.ip` | |
| `src_port` | `src_endpoint.port` | |
| `dest` / `dest_ip` | `dst_endpoint.ip` | |
| `dest_port` | `dst_endpoint.port` | |
| `protocol` | `connection_info.protocol_name` | |
| `bytes_in`/`bytes_out` | `traffic.bytes_in`/`traffic.bytes_out` | (where promoted) |
| `action` (allow/deny/drop) | `disposition`, `disposition_id`, `action`, `action_id` | |
| `user` | `user.name` (User-ID-mapped) | |
| `app` | `app_name` (where promoted) | |
| `rule` | `policy.name` (where promoted) | |

Use `metadata.log_name = 'firewall'` plus `metadata.product.vendor_name = 'Palo Alto Networks'`
(or the silver-table predicate) to distinguish Palo Alto from other firewalls if multiple are
ingested.

## Column mappings — `cisco:asa` → `network_activity`

| Splunk (`cisco:asa`) | Gold `network_activity` | Notes |
| --- | --- | --- |
| `_time` | `time` | |
| `src_ip` / `src` | `src_endpoint.ip` | |
| `src_port` | `src_endpoint.port` | |
| `dest_ip` / `dest` | `dst_endpoint.ip` | |
| `dest_port` | `dst_endpoint.port` | |
| `protocol` | `connection_info.protocol_name` | |
| `action` (Built/Teardown/Deny) | `disposition`, `action` | |
| `message_id` (ASA-N-NNNNNN) | `metadata.event_code` | e.g., `'106023'` for denied connection |

ASA login messages (302013, 302014, 302015, etc.) route to `authentication` instead; HTTP-related
messages go to `http_activity`. Use `metadata.event_code` to discriminate within `cisco:asa`.

## Column mappings — `bro:conn:json` / `zeek:conn:json` → `network_activity`

| Splunk (`bro:conn:json`) | Gold `network_activity` | Notes |
| --- | --- | --- |
| `ts` / `_time` | `time` | |
| `id.orig_h` | `src_endpoint.ip` | Zeek `id.orig_h` is the originator host |
| `id.orig_p` | `src_endpoint.port` | |
| `id.resp_h` | `dst_endpoint.ip` | |
| `id.resp_p` | `dst_endpoint.port` | |
| `proto` | `connection_info.protocol_name` | |
| `service` | `app_name` (where promoted) — usually `ssh`/`dns`/`http`/`tls` | |
| `duration` | `duration` | seconds in Zeek; check OCSF unit |
| `orig_bytes` / `resp_bytes` | `traffic.bytes_in` / `traffic.bytes_out` | |
| `conn_state` | `status_detail` (e.g., `S0`, `S1`, `SF`, `REJ`) | |
| `uid` | `metadata.uid` | Zeek connection uid (used to join across conn/dns/http logs) |

Use `metadata.log_name = 'zeek_conn'` (and `zeek_dns`, `zeek_http`, etc. for the related logs).
Zeek's `uid` is a useful cross-table join key — it's preserved in `metadata.uid`.
