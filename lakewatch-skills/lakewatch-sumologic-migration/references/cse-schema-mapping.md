# Sumo Logic Cloud SIEM (CSE) normalized schema → Lakewatch OCSF gold

**Cloud SIEM (CSE) is a separate product from Sumo core log search.** CSE ingests raw logs and
maps them to a **normalized record schema** — a standard set of attribute names that detection
rules run against regardless of the source vendor. Because CSE records are *already normalized*,
translating a CSE query to Lakewatch OCSF is mostly a **field rename** (both are normalization
layers with similar goals), plus picking the right gold table.

If the query references fields like `srcDevice_ip`, `dstUser_username`, `metadata_vendor`,
`http_url`, `file_hash_sha256`, or the user mentions "Cloud SIEM" / "CSE" / a "rule expression" —
you're on the CSE path. Use this file. For pipe-based `_sourceCategory` log search, use
`sourcecategory-mapping.md` instead.

> **Authoritative field list**: the complete CSE schema lives in the Sumo Cloud SIEM Content
> Catalog — <https://github.com/SumoLogic/cloud-siem-content-catalog> (the `schema/` folder). The
> naming *pattern* below is reliable; check individual attribute names against the catalog when a
> field isn't listed here.

## CSE naming conventions

CSE attribute names follow a consistent pattern — knowing the pattern lets you map fields that
aren't in the tables below:

- **`object_attribute`** (camelCase after an object prefix): `device_ip`, `device_hostname`,
  `user_username`, `user_email`, `http_url`, `dns_query`, `file_path`, `file_hash_sha256`.
- **Directional `src`/`dst` prefixes** for the two ends of an event — these are the most important
  to get right, because swapping them inverts the query:
  - Source side: `srcDevice_ip`, `srcDevice_hostname`, `srcDevice_mac`, `srcUser_username`,
    `srcPort`
  - Destination side: `dstDevice_ip`, `dstDevice_hostname`, `dstDevice_mac`, `dstUser_username`,
    `dstPort`
  - Non-directional / primary entity: `device_ip`, `user_username`
- **`metadata_*`** describe the record/source itself: `metadata_vendor`, `metadata_product`,
  `metadata_deviceEventId`, `metadata_recordType`, `metadata_schema`.
- **Common normalized semantics**: `action`, `severity`, `success`, `bytes`, `bytesIn`,
  `bytesOut`.

## Picking the destination gold table

CSE doesn't have one flat table — records have a **record type** (`metadata_recordType`, e.g.
`Authentication`, `Network`, `Endpoint`, `Audit`) plus the vendor/product. Route to the OCSF gold
table by what the record *is*:

| CSE record character | OCSF gold table | `class_uid` |
| --- | --- | --- |
| Authentication / logon (`metadata_recordType` ~ `Authentication`) | `authentication` | 3002 |
| Network connection / flow / firewall | `network_activity` | 4001 |
| DNS query | `dns_activity` | 4003 |
| HTTP / proxy / web | `http_activity` | 4002 |
| Process launch (Endpoint) | `process_activity` | 1007 |
| File create/modify/delete (Endpoint) | `file_activity` | 1001 |
| Cloud/API control-plane call (`Audit`) | `api_activity` | 6003 |
| Account create/modify/delete | `account_change` | 3001 |
| Group membership change | `group_management` | 3006 |
| Vendor alert / detection / signal | `detection_finding` | 2004 |
| Email | `email_activity` | 4009 |

Then filter by source with `metadata.log_name` (the Lakewatch preset), mapping from
`metadata_vendor` + `metadata_product`.

## CSE attribute → OCSF field mapping (general rules)

These apply across record types:

| CSE attribute | OCSF path | Notes |
| --- | --- | --- |
| `timestamp` / `_messageTime` | `time` | |
| `metadata_vendor` | `metadata.product.vendor_name` | |
| `metadata_product` | `metadata.product.name` | also maps (via preset) to `metadata.log_name` for filtering |
| `metadata_deviceEventId` | `metadata.event_code` | the source's event/action code |
| `metadata_recordType` | (implicit in the gold table choice) | routes the table, no direct column |
| `device_ip` | `device.ip` (endpoint) or `src_endpoint.ip` (network, when non-directional) | context-dependent |
| `device_hostname` | `device.hostname` | |
| `device_mac` | `device.mac` | |
| `srcDevice_ip` | `src_endpoint.ip` | **directional — don't swap** |
| `srcDevice_hostname` | `src_endpoint.hostname` | |
| `srcPort` | `src_endpoint.port` | |
| `dstDevice_ip` | `dst_endpoint.ip` | **directional — don't swap** |
| `dstDevice_hostname` | `dst_endpoint.hostname` | |
| `dstPort` | `dst_endpoint.port` | |
| `user_username` | `user.name` (target) | |
| `user_email` | `user.email_addr` | |
| `srcUser_username` | `actor.user.name` | the initiating user |
| `dstUser_username` | `user.name` | the target user |
| `action` | `activity_name` / `disposition` / `status` (depends on event) | see per-type notes |
| `success` (bool) | `status_id` (1=Success, 2=Failure), `status` | |
| `severity` | `severity` / `severity_id` | |
| `bytesIn` / `bytesOut` | `traffic.bytes_in` / `traffic.bytes_out` | |
| `network_protocol` | `connection_info.protocol_name` | |

## Per-record-type field maps

### Authentication → `authentication`

| CSE | OCSF `authentication` | Notes |
| --- | --- | --- |
| `user_username` / `dstUser_username` | `user.name` | the account authenticating |
| `srcUser_username` | `actor.user.name` | initiator, when distinct |
| `srcDevice_ip` | `src_endpoint.ip` | where the logon came from |
| `dstDevice_hostname` | `dst_endpoint.hostname` | host receiving the logon |
| `success` | `status` / `status_id` (1/2) | |
| `action` | `activity_name` (Logon/Logoff) | |
| `authProtocol` / `authenticationMethod` | `auth_protocol` | where promoted |

### Network → `network_activity`

| CSE | OCSF `network_activity` |
| --- | --- |
| `srcDevice_ip` / `srcPort` | `src_endpoint.ip` / `src_endpoint.port` |
| `dstDevice_ip` / `dstPort` | `dst_endpoint.ip` / `dst_endpoint.port` |
| `network_protocol` | `connection_info.protocol_name` |
| `action` (allowed/blocked/dropped) | `disposition` / `disposition_id` / `action` |
| `bytesIn` / `bytesOut` | `traffic.bytes_in` / `traffic.bytes_out` |
| `bytes` | `traffic.bytes` |

### DNS → `dns_activity`

| CSE | OCSF `dns_activity` |
| --- | --- |
| `dns_query` | `query.hostname` |
| `dns_queryType` | `query.type` |
| `dns_replyCode` | `rcode` |
| `srcDevice_ip` / `dstDevice_ip` | `src_endpoint.ip` / `dst_endpoint.ip` |

### HTTP / Web → `http_activity`

| CSE | OCSF `http_activity` |
| --- | --- |
| `http_url` | `http_request.url` |
| `http_method` | `http_request.http_method` |
| `http_userAgent` | `http_request.user_agent` |
| `http_response_statusCode` | `http_response.code` |
| `srcDevice_ip` / `dstDevice_ip` | `src_endpoint.ip` / `dst_endpoint.ip` |
| `user_username` | `actor.user.name` |

### Endpoint process → `process_activity`

| CSE | OCSF `process_activity` |
| --- | --- |
| `process_commandLine` | `process.cmd_line` |
| `process_name` | `process.name` |
| `process_path` / `process_exe` | `process.file.path` |
| `process_pid` | `process.pid` |
| `parentProcess_name` | `actor.process.name` |
| `parentProcess_commandLine` | `actor.process.cmd_line` |
| `user_username` | `process.user.name` |
| `device_hostname` | `device.hostname` |
| `file_hash_sha256` | `process.file.hashes` (SHA-256 entry in the array) |

### Endpoint file → `file_activity`

| CSE | OCSF `file_activity` |
| --- | --- |
| `file_path` | `file.path` |
| `file_name` | `file.name` |
| `file_hash_sha256` / `file_hash_md5` | `file.hashes` (array of `{algorithm, value}`) |
| `action` (created/modified/deleted) | `activity_id` / `activity_name` |
| `user_username` | `actor.user.name` |
| `process_name` | `actor.process.name` |

### Detection / Signal → `detection_finding`

| CSE | OCSF `detection_finding` |
| --- | --- |
| `threat_name` / `signalName` | `finding_info.title` |
| `severity` | `severity` / `severity_id` |
| `action` (blocked/quarantined) | `disposition` / `disposition_id` |
| `device_hostname` | `device.hostname` |
| `user_username` | `actor.user.name` |

## Hashes are arrays, not flat columns

CSE `file_hash_sha256` is a scalar; OCSF `file.hashes` / `process.file.hashes` is an
**array of structs**. Translate a hash filter with `EXISTS` or `array_contains`:

```sql
-- CSE: file_hash_sha256 = "<sha256>"
WHERE EXISTS (
  SELECT 1 FROM EXPLODE(file.hashes) AS h
  WHERE h.algorithm = 'SHA-256' AND h.value = '<sha256>'
)
```

## Caveats

- **`success` boolean vs `status_id` int.** CSE `success=true/false` collapses to OCSF
  `status_id` (1/2) and `status` ('Success'/'Failure'). Don't compare `status = true`.
- **`action` is overloaded.** In CSE `action` can mean the activity (Logon), the disposition
  (blocked), or the result — the right OCSF target depends on the record type. Check the per-type
  map above.
- **Verify vendor → preset.** CSE `metadata_vendor`/`metadata_product` tells you the source, but
  the Lakewatch preset (`metadata.log_name`) that ingests it may differ in naming. Cross-check
  with `sourcecategory-mapping.md` and `DESCRIBE` the gold table.
