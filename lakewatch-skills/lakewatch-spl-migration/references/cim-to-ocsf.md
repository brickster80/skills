# Splunk CIM datamodels → Lakewatch OCSF gold tables

The **Splunk Common Information Model (CIM)** is a normalization layer that maps vendor-specific
fields to a common schema. Splunk Enterprise Security and most apps query CIM datamodels rather
than raw sourcetypes.

CIM and OCSF have similar goals but different vocabularies. This file maps each CIM datamodel
to the matching Lakewatch OCSF gold table, plus the field-level renames you'll typically need.

CIM and OCSF differ in three structural ways worth knowing up front:

- **CIM is flat; OCSF nests.** `src` (flat string in CIM) → `src_endpoint.ip` (nested struct in
  OCSF). `user` (flat) → `user.name` (under `user` struct).
- **CIM uses string enums; OCSF often pairs them with integer IDs.** CIM `action="success"` →
  OCSF `status='Success'` + `status_id=1`. CIM `vendor_action` → `metadata.event_code`.
- **CIM datamodel nodes don't always map 1:1 to OCSF classes.** Some CIM datamodels (Endpoint,
  Change, Web) cover multiple OCSF classes; you have to pick the right one based on which CIM
  *node* the search uses.

## Datamodel → OCSF class index

| CIM datamodel (root) | Common nodes | OCSF gold class | `class_uid` |
| --- | --- | --- | --- |
| `Authentication` | (root) | `authentication` | 3002 |
| `Network_Traffic.All_Traffic` | `All_Traffic`, `Allowed`, `Blocked`, `IDS_Attacks` | `network_activity` | 4001 |
| `Network_Sessions.All_Sessions` | `Session_Start`, `Session_End` | `network_activity` | 4001 |
| `Network_Resolution.DNS` | `Resolution` | `dns_activity` | 4003 |
| `Web.Web` | `Proxy`, `IDS_Attacks` | `http_activity` | 4002 |
| `Endpoint.Processes` | `Processes` | `process_activity` | 1007 |
| `Endpoint.Filesystem_Changes` | `Filesystem_Changes` | `file_activity` | 1001 |
| `Endpoint.Registry_Changes` | `Registry_Changes` | `entity_management` | 3004 |
| `Endpoint.Ports` | `Ports` | `network_activity` | 4001 |
| `Endpoint.Services` | `Services` | `entity_management` | 3004 |
| `Email.All_Email` | `Delivery`, `Filtering`, `Content` | `email_activity` | 4009 |
| `Change.All_Changes` | `Account_Management`, `Endpoint_Changes`, `Network_Changes` | `account_change` (3001) / `entity_management` (3004) / `group_management` (3006) depending on `change_type` | varies |
| `Malware.Malware_Attacks` | (root) | `detection_finding` | 2004 |
| `Vulnerabilities.Vulnerabilities` | (root) | `vulnerability_finding` | 2002 |
| `Alerts.Alerts` | (root) | `detection_finding` (or `incident_finding` for ES notables) | 2004 / 2005 |
| `Inventory.All_Inventory` | various | **silver only** — Lakewatch holds asset/identity inventory in silver tables (e.g., `mde_device_info`, `mde_identity_info`) rather than gold. Surface as a translation caveat. | — |
| `Performance` | various | **out of scope** — Lakewatch is security-focused; performance metrics live in Databricks system tables. | — |

## CIM → OCSF field mapping (general rules)

These apply across most datamodels:

| CIM field | OCSF path | Notes |
| --- | --- | --- |
| `_time` | `time` | |
| `vendor` | `metadata.product.vendor_name` | |
| `product` | `metadata.product.name` | |
| `vendor_product` | `metadata.product.name` (often the same value as `product`) | |
| `tag` | filter via `metadata.log_name` or the gold table itself — CIM tags don't have a 1:1 OCSF column | |
| `sourcetype` | `metadata.log_name` (closest analog) | |
| `host` | varies — `device.hostname`, `src_endpoint.hostname`, or `dst_endpoint.hostname` depending on the event type | |
| `user` | `user.name` (when the user is the target/subject) **or** `actor.user.name` (when the user is the initiator) | the role distinction is the most common CIM→OCSF confusion |
| `user_id` | `user.uid` / `actor.user.uid` | |
| `src` / `src_ip` | `src_endpoint.ip` | |
| `src_port` | `src_endpoint.port` | |
| `src_user` | `actor.user.name` | the user behind the source endpoint |
| `dest` / `dest_ip` | `dst_endpoint.ip` | |
| `dest_port` | `dst_endpoint.port` | |
| `dest_user` / `target_user` | `user.name` (under the user struct of the gold table) | |
| `app` / `application` | `app_name` (where promoted) or part of `metadata.product.feature.name` | |
| `protocol` | `connection_info.protocol_name` | |
| `transport` | `connection_info.protocol_name` (TCP/UDP) | |
| `action` (`success`, `failure`, `allowed`, `blocked`, …) | drives `status` / `status_id` / `disposition` / `disposition_id` depending on the event semantics | |
| `result` | `status_detail` (when more granular than `action`) | |
| `signature` / `signature_id` | `metadata.event_code` / `finding_info.title` | depends on context |
| `severity` | `severity` / `severity_id` | |
| `bytes_in` / `bytes_out` | `traffic.bytes_in` / `traffic.bytes_out` (where promoted) | |
| `dvc` | `device.hostname` or `device.uid` | the device reporting the event |

## Per-datamodel field maps

### `Authentication` → `authentication`

| CIM (`Authentication`) | OCSF `authentication` | Notes |
| --- | --- | --- |
| `Authentication.user` | `user.name` | the target of the authentication |
| `Authentication.src_user` | `actor.user.name` | the initiator (rare; usually the same as `user`) |
| `Authentication.user_id` | `user.uid` | |
| `Authentication.src` | `src_endpoint.ip` | originator IP |
| `Authentication.dest` | `dst_endpoint.hostname` or `dst_endpoint.ip` | the host receiving the logon |
| `Authentication.action` (`success`/`failure`) | `status` / `status_id` (1=Success, 2=Failure) | |
| `Authentication.app` | `actor.app_name` | for app-based auth (OAuth/SAML), this is the application |
| `Authentication.authentication_method` | `auth_protocol` / `mfa.factor_name` | password / certificate / MFA, etc. |
| `Authentication.authentication_service` | `service.name` | the IdP / domain controller / etc. |
| `Authentication.reason` | `status_detail` | failure reason |

### `Network_Traffic.All_Traffic` → `network_activity`

| CIM | OCSF `network_activity` |
| --- | --- |
| `All_Traffic.src` | `src_endpoint.ip` |
| `All_Traffic.src_port` | `src_endpoint.port` |
| `All_Traffic.dest` | `dst_endpoint.ip` |
| `All_Traffic.dest_port` | `dst_endpoint.port` |
| `All_Traffic.protocol` | `connection_info.protocol_name` |
| `All_Traffic.transport` | `connection_info.protocol_name` (TCP/UDP) |
| `All_Traffic.action` (`allowed`/`blocked`/`dropped`) | `disposition`, `disposition_id`, `action`, `action_id` |
| `All_Traffic.bytes_in`/`bytes_out` | `traffic.bytes_in` / `traffic.bytes_out` |
| `All_Traffic.packets_in`/`packets_out` | `traffic.packets_in` / `traffic.packets_out` |
| `All_Traffic.duration` | `duration` |
| `All_Traffic.app` | `app_name` |
| `All_Traffic.rule` | `policy.name` |
| `All_Traffic.session_id` | `session.uid` (or `metadata.correlation_uid`) |
| `All_Traffic.user` | `user.name` (User-ID mapped) |

### `Network_Resolution.DNS` → `dns_activity`

| CIM | OCSF `dns_activity` |
| --- | --- |
| `DNS.query` | `query.hostname` |
| `DNS.query_type` | `query.type` |
| `DNS.record_type` | `answers[].type` (per-answer) |
| `DNS.answer` | `answers[].rdata` |
| `DNS.reply_code` | `rcode` |
| `DNS.reply_code_id` | `rcode_id` |
| `DNS.src` / `DNS.dest` | `src_endpoint.ip` / `dst_endpoint.ip` |

### `Web.Web` → `http_activity`

| CIM | OCSF `http_activity` |
| --- | --- |
| `Web.url` | `http_request.url` (this is the most-used path in current Lakewatch presets) |
| `Web.http_method` | `http_request.http_method` |
| `Web.http_user_agent` | `http_request.user_agent` |
| `Web.http_referrer` | `http_request.referrer` |
| `Web.status` | `http_response.code` |
| `Web.src` / `Web.dest` | `src_endpoint.ip` / `dst_endpoint.ip` |
| `Web.user` | `actor.user.name` (the user behind the HTTP request) |
| `Web.bytes_in` / `Web.bytes_out` | `traffic.bytes_in` / `traffic.bytes_out` |
| `Web.action` (`allowed`/`blocked`) | `disposition`, `disposition_id` |

### `Endpoint.Processes` → `process_activity`

| CIM | OCSF `process_activity` |
| --- | --- |
| `Processes.process` | `process.cmd_line` (often) OR `process.name` if it's just the binary name |
| `Processes.process_name` | `process.name` |
| `Processes.process_id` | `process.pid` |
| `Processes.process_exec` | `process.file.name` |
| `Processes.process_path` | `process.file.path` |
| `Processes.process_hash` / `Processes.hash` | `process.file.hashes` (array of `{algorithm, value}` structs) |
| `Processes.parent_process` | `actor.process.cmd_line` (or `actor.process.name`) |
| `Processes.parent_process_name` | `actor.process.name` |
| `Processes.parent_process_id` | `actor.process.pid` |
| `Processes.user` | `process.user.name` (the user the process ran as) |
| `Processes.dest` (the endpoint) | `device.hostname` / `device.uid` |

### `Endpoint.Filesystem_Changes` → `file_activity`

| CIM | OCSF `file_activity` |
| --- | --- |
| `Filesystem_Changes.file_name` | `file.name` |
| `Filesystem_Changes.file_path` | `file.path` |
| `Filesystem_Changes.file_hash` | `file.hashes` (array of structs) |
| `Filesystem_Changes.action` (`created`/`modified`/`deleted`) | `activity_id` (1/3/4) / `activity_name` |
| `Filesystem_Changes.user` | `actor.user.name` |
| `Filesystem_Changes.process` / `Filesystem_Changes.process_name` | `actor.process.cmd_line` / `actor.process.name` |
| `Filesystem_Changes.dest` | `device.hostname` |

### `Endpoint.Registry_Changes` → `entity_management`

| CIM | OCSF `entity_management` |
| --- | --- |
| `Registry_Changes.registry_path` | `entity.uid` (the key path) or in `unmapped` depending on preset |
| `Registry_Changes.registry_value_name` | `entity.name` |
| `Registry_Changes.registry_value_data` | `entity.data` (where promoted) |
| `Registry_Changes.action` (`created`/`modified`/`deleted`) | `activity_id` / `activity_name` |
| `Registry_Changes.process` | `actor.process.name` |

### `Email.All_Email` → `email_activity`

| CIM | OCSF `email_activity` | Notes |
| --- | --- | --- |
| `All_Email.recipient` | `email.to` | scalar string in current presets |
| `All_Email.sender` / `All_Email.src_user` | `email.from` **only when the preset promotes it** — many presets keep the from-address in `unmapped` (e.g., the Defender preset). Verify. |
| `All_Email.subject` | `message` **(NOT `email.subject`)** — at least the Defender preset writes the subject to OCSF `message`. Verify per preset. |
| `All_Email.size` | `email.size` (when promoted) |
| `All_Email.message_id` | `metadata.correlation_uid` or `message_trace_uid` (Defender preset writes both) |
| `All_Email.action` | `disposition` / `disposition_id` |
| `All_Email.delay` | `email.delivered_time` minus `time` — derived |

### `Change.All_Changes` → routed by `change_type`

| CIM `change_type` | Gold table |
| --- | --- |
| `AAA` (authentication, authorization, accounting) | `account_change` (user mgmt) or `user_access` (role/permission) |
| `account` | `account_change` |
| `audit` | depends on what was audited |
| `directory_service` | `entity_management` |
| `endpoint` | `entity_management` |
| `filesystem` | `file_activity` (though typically routed via `Endpoint.Filesystem_Changes`) |
| `group` | `group_management` |
| `network` | `entity_management` |
| `registry` | `entity_management` |
| `service` | `entity_management` |

Within each, common fields:

| CIM `All_Changes` | OCSF |
| --- | --- |
| `All_Changes.object` | `entity.name` (for `entity_management`) / `user.name` (for `account_change`) / `group.name` (for `group_management`) |
| `All_Changes.object_id` | `entity.uid` / `user.uid` / `group.uid` |
| `All_Changes.action` (`created`/`updated`/`deleted`) | `activity_id` / `activity_name` |
| `All_Changes.user` (who made the change) | `actor.user.name` |
| `All_Changes.result` | `status` |

### `Malware.Malware_Attacks` → `detection_finding`

| CIM | OCSF `detection_finding` |
| --- | --- |
| `Malware_Attacks.signature` | `finding_info.title` (or sometimes `metadata.event_code`) |
| `Malware_Attacks.category` | `finding_info.types[]` |
| `Malware_Attacks.severity` | `severity` / `severity_id` |
| `Malware_Attacks.file_name` / `file_path` / `file_hash` | embedded in `evidences[]` (as observables) |
| `Malware_Attacks.user` | `actor.user.name` |
| `Malware_Attacks.dest` | the affected device — `device.hostname` |
| `Malware_Attacks.action` (`allowed`/`blocked`/`quarantined`/`deleted`) | `disposition` / `disposition_id` |

### `Vulnerabilities.Vulnerabilities` → `vulnerability_finding`

| CIM | OCSF `vulnerability_finding` |
| --- | --- |
| `Vulnerabilities.signature` / `signature_id` | `vulnerabilities[].title` / `vulnerabilities[].cve.uid` |
| `Vulnerabilities.severity` | `severity` / `severity_id` |
| `Vulnerabilities.cve` | `vulnerabilities[].cve.uid` |
| `Vulnerabilities.cvss` | `vulnerabilities[].cve.cvss[].base_score` |
| `Vulnerabilities.dest` | affected device — `device.hostname` |

## When the SPL uses CIM nodes you don't recognize

CIM has many extension models (Asset_And_Identity_Management, Compute_Inventory, Updates, etc.).
If the SPL references a datamodel that isn't in this file, check the user's Splunk app set
(`btool datamodels list` output is the authoritative answer) and ask which underlying source the
datamodel is built on. Often it's a wrapper around a sourcetype that the
`references/sourcetype-mapping.md` file already covers.
