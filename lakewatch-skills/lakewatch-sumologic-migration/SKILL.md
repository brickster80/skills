---
name: lakewatch-sumologic-migration
description: >
  Translate Sumo Logic queries into Databricks SQL for Databricks Lakewatch. Use this skill
  whenever the user asks to migrate, port, rewrite, convert, or translate Sumo Logic searches
  into SQL — including any mention of Sumo Logic log search, Sumo pipe queries, Sumo
  dashboards/scheduled searches, Sumo Cloud SIEM (CSE) rules, CSE normalized-schema records
  (`srcDevice_ip`, `dstDevice_ip`, `user_username`, …), Sumo metadata fields (`_sourceCategory`,
  `_source`, `_sourceName`, `_sourceHost`, `_collector`, `_index`, `_view`, `_messageTime`,
  `_receiptTime`), Sumo operators (`parse`, `parse regex`, `json`, `keyvalue`/`kv`, `where`,
  `count`, `count_distinct`, `count_frequent`, `values`, `timeslice`, `_timeslice`, `dedup`,
  `sort`, `top`, `limit`, `if`, `format`, `formatDate`, `geoip`, `join`, `lookup`, `transaction`,
  `sessionize`, `accum`, `outlier`, …), or moving Sumo detections/dashboards into Lakewatch.
  Use it even if the user does not say the word "skill" or "Lakewatch" — if there is a Sumo Logic
  query involved and the destination is Databricks, this skill applies. The translation produces
  Databricks SQL plus inline mapping notes (chosen target table, operator/field mapping, caveats).
---

# Lakewatch Sumo Logic → SQL migration

This skill translates **Sumo Logic** queries into Databricks SQL that runs on **Databricks
Lakewatch**. Sumo has two very different query surfaces, and the first job is always to tell
them apart:

- **Core log search** — the pipe-based Search Query Language (`_sourceCategory=aws/cloudtrail
  errorCode | parse "user=*" as user | count by user`). This is the common case and is
  structurally close to Splunk SPL.
- **Cloud SIEM (CSE)** — a *separate product* with its own **normalized record schema**
  (`srcDevice_ip`, `dstUser_username`, `metadata_vendor`, …) and its own rule/record-search
  surface. CSE records are already normalized, which usually makes them a near-direct rename to
  OCSF gold — but the field names are Sumo's, not OCSF's.

The goal is a translation that:

1. Runs unmodified against Lakewatch tables.
2. Uses the **correct Lakewatch destination table** (OCSF gold tables first; silver vendor tables
   only when the gold table cannot represent the data).
3. Comes with **inline mapping notes** so a security engineer can review the choice of table,
   operator translation, and any semantic caveats.

## Why this is harder than a textbook SQL conversion

Sumo log search and Databricks SQL look similar on the surface (both pipe-ish, both have `count
by` analogues), but the translation work lives in three places:

- **Schema is not a rename.** A Sumo `_sourceCategory=aws/cloudtrail` message or a raw syslog line
  does not exist as a table in Lakewatch. Lakewatch normalizes events to a **modified OCSF 1.3.0**
  gold layer (`authentication`, `network_activity`, `process_activity`, …). One `_sourceCategory`
  maps to one or more OCSF classes; many raw fields (which in Sumo you had to `parse` out of
  `_raw`) already exist as typed OCSF struct paths in Lakewatch — so a big chunk of a Sumo query
  (the `parse` / `json` / `keyvalue` extraction pipeline) often collapses to *nothing*, because
  Lakewatch already extracted those fields at ingest.
- **Operators don't all map cleanly.** `count_frequent`, `values`, `timeslice`, `accum`,
  `outlier`, `transaction`/`sessionize`, `most_recent`, `geoip`, `parse regex … multi` need
  careful Spark SQL equivalents — sometimes window functions, `QUALIFY`, `LATERAL VIEW EXPLODE`,
  `COLLECT_SET`, `date_trunc`, or a CTE.
- **CSE vs log-search query styles need different translations.** A CSE record query
  (`srcDevice_ip = "10.0.0.1" and metadata_vendor = "Palo Alto"`) is almost a rename to the OCSF
  path — `srcDevice_ip` → `src_endpoint.ip`, `dstDevice_ip` → `dst_endpoint.ip`. A raw
  log-search query (`_sourceCategory=Palo/Traffic | parse …`) needs `_sourceCategory`-aware
  routing plus operator translation for the whole extraction pipeline.

Read the WHY before forcing a translation — if the source Sumo query depends on Sumo-only behavior
(a `lookup` against a Sumo lookup table that isn't in Lakewatch, a Scheduled View / Partition
`_index=`, a custom `parse` over a field Lakewatch doesn't preserve), surface that in the mapping
notes instead of guessing.

## Workflow

### 1. Read the Sumo query carefully

Before writing any SQL, identify:

- **Which surface is this — log search or CSE?**
  - Pipe operators over `_sourceCategory` / `_source` / `_index`, with `parse`/`json`/`keyvalue`
    extraction → **core log search**. Use `references/sourcecategory-mapping.md`.
  - Field references like `srcDevice_ip`, `dstUser_username`, `metadata_product`, `file_hash_sha256`,
    or the user says "Cloud SIEM" / "CSE" / "rule expression" → **CSE**. Use
    `references/cse-schema-mapping.md`.
- **Source identification (log search)** — what is the query scoped to? `_sourceCategory=foo`,
  `_source=bar`, `_index=<partition_or_view>`, or a bare keyword search? Each implies a different
  Lakewatch preset. `_sourceCategory` is the primary analog to Splunk's `sourcetype` / `index`.
- **Extraction pipeline (log search)** — which `parse` / `parse regex` / `json` / `keyvalue` /
  `csv` / `split` steps create fields? **Most of these disappear on Lakewatch** because the preset
  already promoted those fields to OCSF struct paths. Only keep an extraction step if the field
  the Sumo query pulls out is *not* mapped by the preset (in which case it lives in `unmapped`
  VARIANT or the silver `data` column).
- **Time range** — Sumo searches almost always take their window from the **UI time picker or the
  Search Job API `from`/`to`**, not the query text (see operator reference). Watch for in-query
  `_messageTime` / `_receiptTime` / `queryStartTime()` logic — that's deliberate and must be
  preserved.
- **Aggregations** — what is being `count`'d, `sum`'d, grouped (`by`), or windowed
  (`timeslice`, `accum`).
- **Output shape** — what the final `fields` / `sort` / `top` / aggregation produces (the columns
  consumers see).

If any `_sourceCategory` is unfamiliar, **do not guess its purpose** — ask the user. Source
Categories are customer-defined free-form strings (`prod/aws/cloudtrail`, `Palo/Traffic`,
`OS/Linux/Security`), so their meaning varies per deployment far more than Splunk sourcetypes do.

### 2. Pick the destination table

The decision tree:

1. **Is there a matching OCSF gold table?** This is the strongly preferred target — detections,
   joins, and dashboards built on gold are vendor-neutral.
   - Sumo `_sourceCategory=*/cloudtrail` → `api_activity` (most), `authentication` (ConsoleLogin),
     `account_change` (IAM)
   - Sumo `_sourceCategory=*/palo*/traffic` (or CSE Palo Alto traffic records) → `network_activity`
   - Sumo Windows Security events (`_sourceCategory=*/windows/security`, EventID 4624/4625) →
     `authentication`
   - CrowdStrike / EDR process events → `process_activity`
   - See `references/sourcecategory-mapping.md` for the log-search `_sourceCategory` → Lakewatch
     preset index, and `references/cse-schema-mapping.md` for the CSE normalized-schema → OCSF
     gold mapping. **Read the relevant reference file before picking** — guessing leads to wrong
     `class_uid` filtering.
2. **If no gold mapping exists**, fall back to the **silver vendor table** (e.g., `aws_cloudtrail`,
   `paloalto_traffic`, `crowdstrike_process_rollup`). Silver tables keep more of the raw vendor
   schema and are useful when the Sumo query reads a field that has no clean OCSF equivalent
   (something you'd have `parse`d out of `_raw` in Sumo).
3. **If even silver doesn't exist**, tell the user that data source isn't ingested by Lakewatch
   and stop. Don't invent a target table.

When uncertain, verify against the live catalog rather than guessing — use the `databricks-dbsql`
or `databricks-unity-catalog` skill to `DESCRIBE` the candidate table before committing to the
translation. Schemas evolve and the bundled references can lag.

**Source filter rule of thumb**: gold tables hold events from multiple presets, so almost every
translated query needs a source filter in its `WHERE`. Prefer **`metadata.log_name`** (the
preset's `sourceType` — short string like `cloudtrail`, `firewall`, `windows_events_xml`,
`falcon_per_event_schema`) as the primary filter. It's smaller on disk than
`metadata.product.name`, gets better Delta data-skipping, and is the most stable value across
upgrades. Add `metadata.product.name` only when `log_name` is too coarse. **Do not** try to
reproduce the Sumo `_sourceCategory` string literally — it's a Sumo-side collection label with no
Lakewatch column; the equivalent discriminator is the `metadata.log_*` fields.

### 3. Translate Sumo operators

`references/sumo-operators.md` is the operator-level cheat sheet. Read it whenever the source
query uses anything beyond a scope filter + `count by …`.

Patterns worth knowing up front:

- **Default for Sumo → `INRANGE(time)`** (Lakewatch's bind-to-UI-timespan-picker primitive).
  Sumo's time range comes from the **search UI time picker / API `from`+`to`**, essentially never
  from the query body — even more so than Splunk. So a Sumo query almost always means "run this
  over whatever range the picker has selected," which is exactly what `INRANGE(time)` reproduces
  on the Lakewatch explore view / dashboard. Only translate to a literal `INTERVAL` when the user
  says they're wrapping the result into a **scheduled rule** (rules get no UI picker; the lookback
  has to be concrete and tied to the rule's `schedule.atLeastEvery`) or explicitly wants a hard
  pin. If the query has in-body `_messageTime`/`_receiptTime`/`queryStartTime()` logic, preserve
  that literally — it was deliberate. See `references/sumo-operators.md` → *Time, dates, and
  intervals*.
- **The extraction pipeline usually collapses.** `| parse "user=*" as user`, `| json "srcIp" as
  src`, `| keyvalue auto`, `| parse regex "(?<ip>...)"` — if the preset already promotes that
  field (check `references/sourcecategory-mapping.md`), **delete the extraction step** and use the
  OCSF struct path directly. Only translate the extraction when the field lives in `unmapped`
  (then `variant_get(unmapped, '$.<name>', '<type>')`) or you had to drop to silver (then
  `variant_get(data, '$.<path>', '<type>')`).
- `count by foo` → `SELECT foo, COUNT(*) FROM t GROUP BY foo`
- `count_distinct(x) by foo` → `SELECT foo, COUNT(DISTINCT x) FROM t GROUP BY foo`
- `values(x) by foo` → `SELECT foo, COLLECT_SET(x) AS x FROM t GROUP BY foo`
- `count_frequent by foo` → `SELECT foo, COUNT(*) AS n FROM t GROUP BY foo ORDER BY n DESC`
- `timeslice 1h | count by _timeslice` → `date_trunc('HOUR', time)` in the `GROUP BY`
- `dedup by foo` → `SELECT DISTINCT` or `QUALIFY ROW_NUMBER() OVER (PARTITION BY foo ORDER BY time
  DESC) = 1` for the most-recent row per `foo`
- `if(cond, a, b) as foo` → `IF(cond, a, b) AS foo` (or `CASE WHEN …`) inline in the `SELECT`
- `top 10 foo by _count` → `… GROUP BY foo ORDER BY COUNT(*) DESC LIMIT 10`
- `accum x` (running total) → `SUM(x) OVER (ORDER BY time ROWS BETWEEN UNBOUNDED PRECEDING AND
  CURRENT ROW)`
- `transaction on sessionid` / `sessionize` → CTE + window functions (see operator reference)
- `lookup col from path://"…" on key=field` → `LEFT JOIN <lookup_table> l ON l.key = field`; the
  lookup table has to exist in Lakewatch — ask the user where it lives.
- **matching command lines and CLI switches** (`ProcessCommandLine matches "*-enc*"`, `... has
  "powershell"`) → a textbook `\bword\b` regex is **wrong** for command-line tokens. Read
  `references/sumo-operators.md` → *Strings, parsing, regex* → *Command-line and switch matching*.

Don't introduce a CTE per derived field — the resulting SQL becomes unreadable. Inline expressions
in the `SELECT` are easier to review and Spark optimizes them identically.

### 4. Map field references

After translating operators, map every column reference. The biggest gotchas:

- **The preset is the contract, not the OCSF spec.** Each Lakewatch preset chooses which source
  fields to promote to typed OCSF struct paths and which to keep in the `unmapped` VARIANT column.
  **Read the relevant `references/sourcecategory-mapping.md` row before naming a target path** —
  guessing from OCSF docs alone produces SQL that returns NULL for the field you thought you mapped.
- **CSE → OCSF is mostly a rename**, with a few structural differences. CSE's directional prefixes
  map cleanly: `srcDevice_ip` → `src_endpoint.ip`, `dstDevice_ip` → `dst_endpoint.ip`,
  `srcUser_username` → `actor.user.name`, `dstUser_username`/`user_username` → `user.name`,
  `file_hash_sha256` → the `file.hashes` array. See `references/cse-schema-mapping.md`.
- **Sumo log-search fields are whatever you parsed; OCSF gold puts them in nested structs.** A
  field you'd `parse "user=*" as user` becomes `user.name`. A `src_ip` you extracted becomes
  `src_endpoint.ip`. A `process` string becomes `process.name` (or `process.cmd_line` if it's a
  full command line).
- **Sumo `_messageTime` is `time` in every Lakewatch silver and gold table.** Don't preserve
  `_messageTime`. `_receiptTime` maps roughly to `metadata.logged_time` / `metadata.processed_time`.
- **Sumo `_raw` does not exist on gold.** The closest equivalent is `raw_data` (VARIANT) preserved
  by some presets, or you drop down to the silver table's `data` column.
- **Sumo metadata fields** (`_sourceCategory`, `_source`, `_sourceHost`, `_collector`) don't have
  direct OCSF homes — see the mapping table in `references/sourcecategory-mapping.md`. `_sourceHost`
  usually becomes `device.hostname` / `src_endpoint.hostname` depending on the event role.

Field-level mappings live in:

- `references/sourcecategory-mapping.md` — Sumo `_sourceCategory` values → Lakewatch presets +
  column maps, plus the Sumo metadata-field table.
- `references/cse-schema-mapping.md` — Sumo Cloud SIEM normalized schema → OCSF gold class + field
  mappings.
- `references/ocsf-gold-tables.md` — Concise list of Lakewatch OCSF gold tables and their
  `class_uid`. Shared with the SPL/KQL skills (symlinked).

### 5. Write the output

Produce **two parts**:

**Part A — the translated SQL.** Runnable as-is. Use lowercase keywords or uppercase consistently
(match the user's existing style if they have one in scope). Place a `-- ` header comment at the
top naming the destination table and the original `_sourceCategory` / CSE source.

**Part B — mapping notes.** Below the SQL, in a fenced block titled `Mapping notes`, include:

- The **destination table** chosen and **why** (which OCSF class it maps to).
- A short **column mapping table** showing source Sumo field → SQL expression. Only include rows
  where the mapping is non-obvious (renames, struct lookups, type casts, enum collapses, and
  especially any extraction step that collapsed because the preset already promoted the field).
- Any **semantic caveats** — fields that don't have a clean equivalent, lookback assumptions,
  joins/lookups that had to be restructured, CSE↔OCSF differences.
- If you fell back to a silver table or had to skip a piece of the query, **say so explicitly**.

### Example output shape

```sql
-- Source: Sumo Logic `_sourceCategory=*/cloudtrail eventName=ConsoleLogin errorMessage="Failed authentication"`
--         — failed AWS console logins by user (time from the search-bar picker)
-- Destination: <catalog>.gold.authentication (Lakewatch OCSF, fed by the cloudtrail preset)
SELECT
  user.name AS user_name,
  COUNT(*) AS failed_attempts,
  COLLECT_SET(src_endpoint.ip) AS src_ips
FROM gold.authentication
WHERE INRANGE(time)
  AND metadata.log_name = 'cloudtrail'
  AND metadata.event_code = 'ConsoleLogin'
  AND status = 'Failure'
GROUP BY user.name
HAVING COUNT(*) >= 5
ORDER BY failed_attempts DESC
```

**Mapping notes**

- **Destination**: `authentication` (OCSF class 3002), fed by the `cloudtrail` preset. CloudTrail
  `ConsoleLogin` events route to `authentication`, *not* `api_activity` (which holds the rest of
  CloudTrail). Filter via `metadata.event_code = 'ConsoleLogin'`.
- **Source filter**: `metadata.log_name = 'cloudtrail'` isolates CloudTrail from the other
  sources that also land in `authentication`. The Sumo `_sourceCategory` string itself has no
  Lakewatch column — `metadata.log_name` is the equivalent discriminator.
- **Extraction collapsed**: the Sumo original would typically `| json "userIdentity.userName" as
  user | json "sourceIPAddress" as src_ip`. On Lakewatch the `cloudtrail` preset already promotes
  these, so those steps are gone — use `user.name` and `src_endpoint.ip` directly.
- **Time range**: Sumo takes its window from the search picker / API `from`+`to`, so I used
  `INRANGE(time)` to bind to the Lakewatch explore-view timespan picker. If this is going into a
  scheduled rule, swap to `time >= current_timestamp() - INTERVAL <n> HOUR`.
- **Catalog**: the Lakewatch catalog name varies by environment (`lw-prod-catalog`,
  `lakewatch_<env>`, a sandbox name). Many Lakewatch SQL contexts default to `gold` as the active
  schema, so the catalog/schema prefix is often optional.
- **Column mapping**:

  | Sumo | Lakewatch `authentication` |
  | --- | --- |
  | `_messageTime` | `time` |
  | `userIdentity.userName` (parsed) | `user.name` |
  | `sourceIPAddress` (parsed) | `src_endpoint.ip` |
  | `eventName` | `metadata.event_code` |
  | `errorMessage="Failed authentication"` | `status = 'Failure'` |

- **Caveat**: Sumo `values(sourceIPAddress)` returns up to 100 distinct values lexicographically;
  `COLLECT_SET` returns all distinct values unordered. If you need the 100-value cap, add
  `SLICE(ARRAY_SORT(COLLECT_SET(...)), 1, 100)`.

## When the user wants more than a translation

If the user is **building a detection rule** out of the translated query, route them to the
`lakewatch-rule-dev` skill once translation is done — that skill handles the YAML wrapping, MITRE
mapping, output template, and validator workflow.

If the user is **building a dashboard widget**, point them at `lakewatch-dashboard-dev`.

This skill stops at "here is the SQL + mapping notes." Don't try to also produce a rule YAML in
the same pass — it makes review harder and the rule skill has its own validation steps.

## Common mistakes to avoid

- **Don't carry the `parse`/`json`/`keyvalue` pipeline over verbatim.** The single most common
  mistake is faithfully translating Sumo's field-extraction steps into `regexp_extract` /
  `variant_get` when Lakewatch already promoted that field to an OCSF struct path at ingest. Check
  the preset mapping first — most extraction collapses to a direct column reference.
- **Don't try to filter on `_sourceCategory`.** There is no `_sourceCategory` column in Lakewatch.
  Use `metadata.log_name` (and `metadata.product.name` when needed) as the source discriminator.
- **Don't invent column names.** If unsure whether OCSF `authentication` has `user.uid` or
  `user.id`, check `references/cse-schema-mapping.md`, the OCSF docs, or `DESCRIBE` the table.
- **Don't strip the time filter.** Lakewatch tables can be very large. Sumo's missing in-query
  time clause means it relied on the UI/API picker — translate that with `INRANGE(time)` for
  interactive contexts, or a sensible default (7 days) for scheduled rules, and surface the
  assumption.
- **Don't translate `lookup` / `save` / `_index=<view>` / custom Sumo lookup tables silently.**
  These reference Sumo-only storage; flag them and ask the user where the equivalent data lives in
  Lakewatch.
- **Don't confuse `count` and `count_distinct`.** Sumo `count_distinct(x)` → `COUNT(DISTINCT x)`,
  not `COUNT(x)`. Sumo `count` → `COUNT(*)`; `count(field)` counts non-null values → `COUNT(field)`.
- **Don't backquote OCSF struct paths.** `user.name` is correct; `` `user.name` `` makes it a flat
  column lookup that will fail.
- **Don't treat CSE `src`/`dst` prefixes as interchangeable.** `srcDevice_ip` → `src_endpoint.ip`
  and `dstDevice_ip` → `dst_endpoint.ip` are directional; swapping them inverts the query meaning.

## Reference files

- `references/sumo-operators.md` — Sumo operator → Spark SQL translations with examples.
- `references/sourcecategory-mapping.md` — Sumo `_sourceCategory` values → Lakewatch presets +
  column mappings, plus the Sumo metadata-field table.
- `references/cse-schema-mapping.md` — Sumo Cloud SIEM normalized schema → OCSF gold class + field
  mappings.
- `references/ocsf-gold-tables.md` — Concise list of Lakewatch OCSF gold tables and their
  `class_uid`. **Symlinked** from the `lakewatch-kql-migration` skill; edit in one place.
