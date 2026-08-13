---
name: lakewatch-spl-migration
description: >
  Translate Splunk Search Processing Language (SPL) into Databricks SQL for Databricks Lakewatch.
  Use this skill whenever the user asks to migrate, port, rewrite, convert, or translate SPL into
  SQL — including any mention of Splunk searches, SPL pipelines, Splunk dashboards/savedsearches,
  Splunk Enterprise Security correlations, ES notable events, Splunk CIM (Common Information
  Model) datamodels (`Authentication`, `Network_Traffic`, `Endpoint.Processes`,
  `Endpoint.Filesystem_Changes`, `Web`, `Email`, `Change`, `Malware`), Splunk sourcetypes
  (`WinEventLog:Security`, `crowdstrike:falconhost:json`, `aws:cloudtrail`, `pan:traffic`,
  `cisco:asa`, `bro:conn:json`, `zscaler:nss`, …), Splunk commands (`stats`, `eval`, `where`,
  `lookup`, `rex`, `dedup`, `transaction`, `tstats`, `streamstats`, `eventstats`, `mvexpand`,
  `spath`, `rename`, `fields`, `bin`, `timechart`), or moving Splunk detections into Lakewatch.
  Use it even if the user does not say the word "skill" or "Lakewatch" — if there is SPL involved
  and the destination is Databricks, this skill applies. The translation produces Databricks SQL
  plus inline mapping notes (chosen target table, operator/field mapping, caveats).
---

# Lakewatch SPL → SQL migration

This skill translates **Splunk Search Processing Language (SPL)** queries into Databricks SQL
that runs on **Databricks Lakewatch**. Both raw-sourcetype SPL (`sourcetype=WinEventLog:Security
EventCode=4625 | stats count by user`) and CIM-normalized SPL (`from datamodel:"Authentication"
where action=failure | stats count by user`) are in scope.

The goal is a translation that:

1. Runs unmodified against Lakewatch tables.
2. Uses the **correct Lakewatch destination table** (OCSF gold tables first; silver vendor tables
   only when the gold table cannot represent the data).
3. Comes with **inline mapping notes** so a security engineer can review the choice of table,
   operator translation, and any semantic caveats.

## Why this is harder than a textbook SQL conversion

SPL and Databricks SQL look similar at first (both pipe-style on the surface, both have `stats`
analogues), but the translation work is in three different places:

- **Schema is not a rename.** A Splunk `sourcetype=aws:cloudtrail` row or a `WinEventLog:Security
  EventCode=4625` row does not exist as a table in Lakewatch. Lakewatch normalizes events to a
  **modified OCSF 1.3.0** gold layer (`authentication`, `network_activity`, `process_activity`,
  …). One sourcetype maps to one or more OCSF classes; many source fields get renamed,
  restructured into nested structs, or fold into enums (e.g., Splunk `vendor_action=success` →
  OCSF `status='Success'` + `status_id=1`).
- **Operators don't all map cleanly.** `stats values(x) as xs by user`, `eval foo = case(...)`,
  `transaction`, `streamstats`, `eventstats`, `mvexpand`, `tstats`, `rex` need careful Spark SQL
  equivalents — sometimes with window functions, `QUALIFY`, `LATERAL VIEW EXPLODE`,
  `regexp_extract`, or a CTE per transaction.
- **CIM vs raw-sourcetype query styles need different translations.** A CIM-normalized search
  (`tag=authentication action=failure`) is almost a rename to the OCSF path — `Authentication`
  datamodel maps to `authentication` gold class, `action` maps to `activity_name`/`status`.
  A raw-sourcetype search (`sourcetype=WinEventLog:Security EventCode=4625 user=…`) needs
  EventID-aware routing because `WinEventLog:Security` covers logon, account-change, group-
  change, file-share, and many other event categories that each go to a different gold table.

Read the WHY before forcing a translation — if the source SPL depends on Splunk-only behavior
(e.g., `lookup` against a CSV that's not in Lakewatch, `inputlookup`, ES `notable` macros, custom
search commands), surface that in the mapping notes instead of guessing.

## Workflow

### 1. Read the SPL carefully

Before writing any SQL, identify:

- **Source identification** — is the query keyed off `sourcetype=…`, `index=…`, a `datamodel:…`
  reference, or a CIM `tag=…`? Each implies a different translation strategy:
  - `sourcetype=foo` / `index=bar` → raw-sourcetype path; you need to know which Lakewatch
    preset ingests that data.
  - `datamodel:Authentication` / `from datamodel "Authentication"` → CIM path; almost a direct
    rename to the matching OCSF gold class.
  - `tag=authentication` (with no datamodel) → CIM-style tag filter on the raw events; treat the
    `tag` as a router hint and pick the matching gold table.
- **Time range** — Splunk searches usually pin time via UI (`earliest=-7d latest=now`) or inline
  (`earliest=-1h`). Watch for searches with **no time clause** — those rely on the dashboard /
  search-bar time picker, which on Lakewatch becomes `INRANGE(time)` (see operator reference).
- **Stats / aggregations** — what is being `stats`'d, grouped, or windowed.
- **Output shape** — what `fields` or final `table` clause produces (the columns that consumers
  see).

If any sourcetype is unfamiliar, **do not guess its purpose** — ask the user or look it up.
Sourcetypes overlap in confusing ways (`cisco:asa` ≠ `cisco:firepower`, `aws:cloudtrail` ≠
`aws:s3:accesslogs`, etc.).

### 2. Pick the destination table

The decision tree:

1. **Is there a matching OCSF gold table?** This is the strongly preferred target — detections,
   joins, and dashboards built on gold are vendor-neutral.
   - `WinEventLog:Security` EventCode 4624/4625 → `authentication`
   - `crowdstrike:falconhost:json` ProcessRollup2 → `process_activity`
   - `aws:cloudtrail` → `api_activity` (most), `authentication` (ConsoleLogin), `account_change`
     (IAM events)
   - `pan:traffic` → `network_activity`
   - `cisco:asa` → `network_activity` (connection events), `authentication` (login events)
   - See `references/sourcetype-mapping.md` for the full Splunk-sourcetype → Lakewatch-preset
     index, and `references/cim-to-ocsf.md` for the CIM datamodel → OCSF gold mapping. **Read
     the relevant reference file before picking** — guessing leads to wrong `class_uid`
     filtering.
2. **If no gold mapping exists**, fall back to the **silver vendor table** (e.g.,
   `crowdstrike_process_rollup`, `cisco_asa`, `paloalto_traffic`). Silver tables keep more of
   the raw vendor schema and are useful when the SPL reads fields that don't have a clean OCSF
   equivalent.
3. **If even silver doesn't exist**, tell the user the sourcetype isn't ingested by Lakewatch
   and stop. Don't invent a target table.

When uncertain, verify against the live catalog rather than guessing — use the `databricks-dbsql`
or `databricks-unity-catalog` skill to `DESCRIBE` the candidate table before committing to the
translation. Schemas evolve and the bundled references can lag.

**Source filter rule of thumb**: gold tables hold events from multiple presets, so almost every
translated query needs a source filter in its `WHERE`. Prefer **`metadata.log_name`** (the
preset's `sourceType` — short string like `cloudtrail`, `falcon_per_event_schema`, `asa`,
`firewall`, `windows_events_xml`) as the primary filter. It's smaller on disk than
`metadata.product.name`, gets better Delta data-skipping, and is the most stable value across
upgrades. Add `metadata.product.name` only when `log_name` is too coarse for the discrimination
you need.

### 3. Translate SPL operators

`references/spl-operators.md` is the operator-level cheat sheet. Read it whenever the source
query uses anything beyond `search` / `where` / `stats count by …`.

Patterns worth knowing up front:

- **Default for SPL → `INRANGE(time)`** (Lakewatch's bind-to-UI-timespan-picker primitive),
  whether or not the source SPL has `earliest=`/`latest=`. Splunk's `earliest=`/`latest=` are
  almost always **placeholder defaults** that the search-bar / dashboard time picker overrides
  in practice — unlike KQL's `ago()` they're rarely deliberate pins. Translating to `INRANGE`
  reproduces that UX on Lakewatch: the explore-view / dashboard timespan picker drives the
  lookback. Surface the literal interval from the SPL in the mapping notes ("the source SPL had
  `earliest=-24h`; switch back to a literal `time >= current_timestamp() - INTERVAL 24 HOUR` if
  you actually want a hard pin").
- **Use a literal `INTERVAL` only when wrapping a scheduled rule.** Rules don't get a UI picker,
  so the lookback has to be concrete and tied to the rule's `schedule.atLeastEvery`. If the user
  says "this is going to be a scheduled detection," translate `earliest=-7d` to
  `current_timestamp() - INTERVAL 7 DAY`. Otherwise prefer `INRANGE(time)`.
- See `references/spl-operators.md` → *Time, dates, and intervals* → *`INRANGE(time)`* for the
  full guidance.
- `stats count by foo` → `SELECT foo, COUNT(*) FROM t GROUP BY foo`
- `stats values(x) as xs by foo` → `SELECT foo, COLLECT_SET(x) AS xs FROM t GROUP BY foo`
- `stats dc(x) by foo` → `SELECT foo, COUNT(DISTINCT x) FROM t GROUP BY foo`
- `dedup foo` → `SELECT DISTINCT foo …` or `QUALIFY ROW_NUMBER() OVER (PARTITION BY foo ORDER BY
  _time DESC) = 1` if you want the most-recent row per `foo`
- `eval foo = case(...)` → `CASE WHEN … THEN … END AS foo` inline in the `SELECT`
- `rex field=x "(?<g>regex)"` → `regexp_extract(x, 'regex', 1) AS g`
- `mvexpand foo` → `LATERAL VIEW EXPLODE(foo)` (or `EXPLODE` in a `SELECT`)
- `transaction sessionid maxspan=30m` → not a one-liner; uses CTE + window functions to
  group/session events. See operator reference.
- `lookup my_lookup field AS x OUTPUT y` → `LEFT JOIN <lookup_table> l ON l.field = x` and
  select `l.y`. The lookup table has to exist in Lakewatch — ask the user where it lives.
- `has` on command lines and switch names (`-EncodedCommand`, `-enc`, `powershell`) → a textbook
  `\bword\b` regex is **wrong** for command-line tokens. Read `references/spl-operators.md` →
  *Strings, parsing, regex* → *Command-line and switch matching*.

Don't introduce a CTE per `eval` step — the resulting SQL becomes unreadable. Inline expressions
in the `SELECT` are easier to review and Spark optimizes them identically.

### 4. Map field references

After translating operators, map every column reference. The biggest gotchas:

- **The preset is the contract, not the OCSF spec.** Each Lakewatch preset chooses which source
  fields to promote to typed OCSF struct paths and which to keep in the `unmapped` VARIANT
  column. **Read the relevant `references/sourcetype-mapping.md` row before naming a target
  path** — guessing from OCSF docs alone produces SQL that returns NULL for the field you thought
  you mapped.
- **CIM → OCSF is mostly a rename**, but with a handful of structural differences. CIM's `src` /
  `dest` (flat strings) become `src_endpoint.ip` / `dst_endpoint.ip` (structs). CIM's `user`
  becomes `user.name` (in `authentication` / `account_change` / etc.) or `actor.user.name` (when
  the user is the initiator, not the target). See `references/cim-to-ocsf.md` for the full
  per-datamodel mapping.
- **Splunk fields are flat; OCSF gold puts them in nested structs.** Splunk's `user` →
  `user.name`. Splunk's `src_ip` → `src_endpoint.ip`. Splunk's `process` → `process.name` (or
  `process.cmd_line` if it's actually a full command line).
- **Splunk `_time` is `time` in every Lakewatch silver and gold table.** Don't preserve `_time`.
- **Splunk `_raw` does not exist on gold.** The closest equivalent is `raw_data` (VARIANT)
  preserved by some presets, or you have to drop down to the silver table's `data` column.

Field-level mappings for the most common sources are in:

- `references/sourcetype-mapping.md` — Splunk sourcetypes → Lakewatch presets + column maps.
- `references/cim-to-ocsf.md` — Splunk CIM datamodels → OCSF gold class + column maps.
- `references/ocsf-gold-tables.md` — Concise list of Lakewatch OCSF gold tables and their
  `class_uid`. Same content as the KQL skill (symlinked).

### 5. Write the output

Produce **two parts**:

**Part A — the translated SQL.** Runnable as-is. Use lowercase keywords or uppercase consistently
(match the user's existing style if they have one in scope). Place a `-- ` header comment at the
top naming the destination table and the original sourcetype/datamodel.

**Part B — mapping notes.** Below the SQL, in a fenced block titled `Mapping notes`, include:

- The **destination table** chosen and **why** (which OCSF class it maps to).
- A short **column mapping table** showing source SPL field → SQL expression. Only include rows
  where the mapping is non-obvious (renames, struct lookups, type casts, enum collapses).
- Any **semantic caveats** — fields that don't have a clean equivalent, lookback assumptions,
  joins that had to be restructured, CIM↔OCSF differences (CIM `success` is a string vs OCSF
  `status_id` is an int, etc.).
- If you fell back to a silver table or had to skip a piece of the query, **say so explicitly**.

### Example output shape

```sql
-- Source: Splunk `sourcetype=WinEventLog:Security EventCode=4625 earliest=-24h` — failed logons by user
-- Destination: <catalog>.gold.authentication (Lakewatch OCSF, fed by the windows_events_xml preset)
SELECT
  user.name AS user_name,
  COUNT(*) AS failed_attempts,
  COUNT(DISTINCT src_endpoint.ip) AS distinct_src_ips
FROM gold.authentication
WHERE INRANGE(time)
  AND metadata.log_name = 'windows_events_xml'
  AND metadata.event_code = '4625'
GROUP BY user.name
HAVING COUNT(*) >= 10
```

**Mapping notes**

- **Destination**: `authentication` (OCSF class 3002), fed by the `windows_events_xml` preset.
  `EventCode=4625` is a logon failure → filter via `metadata.event_code = '4625'` (OCSF lifts
  Windows EventID into `metadata.event_code`).
- **Source filter**: `metadata.log_name = 'windows_events_xml'` isolates Windows Security
  Auditing events from Defender `DeviceLogonEvents` and Entra sign-ins that also land in
  `authentication`.
- **Time range**: the SPL had `earliest=-24h`, but I used `INRANGE(time)` so the Lakewatch
  explore-view / dashboard timespan picker drives the lookback (Splunk's `earliest=` is almost
  always a placeholder default the picker overrides). If this is going into a scheduled rule
  rather than an interactive query, swap to `time >= current_timestamp() - INTERVAL 24 HOUR`.
- **Catalog**: the Lakewatch catalog name varies by environment — common patterns are
  `lw-prod-catalog`, `lakewatch_<env>`, or a sandbox name. Many Lakewatch SQL contexts default
  to `gold` as the active schema, so the catalog/schema prefix is often optional.
- **Column mapping**:

  | Splunk (`WinEventLog:Security`) | Lakewatch `authentication` |
  | --- | --- |
  | `_time` | `time` |
  | `user` | `user.name` |
  | `src_ip` / `IpAddress` | `src_endpoint.ip` |
  | `EventCode` | `metadata.event_code` |

- **Caveat**: SPL `stats dc(src_ip)` is an exact distinct count by default (unlike KQL's `dcount`
  which is HLL); `COUNT(DISTINCT …)` matches.

## When the user wants more than a translation

If the user is **building a detection rule** out of the translated query, route them to the
`lakewatch-rule-dev` skill once translation is done — that skill handles the YAML wrapping,
MITRE mapping, output template, and validator workflow.

If the user is **building a dashboard widget**, point them at `lakewatch-dashboard-dev`.

This skill stops at "here is the SQL + mapping notes." Don't try to also produce a rule YAML in
the same pass — it makes review harder and the rule skill has its own validation steps.

## Common mistakes to avoid

- **Don't invent column names.** If you're unsure whether OCSF `authentication` has `user.uid`
  or `user.id`, check `references/cim-to-ocsf.md`, the OCSF docs, or `DESCRIBE` the table.
- **Don't strip the time filter.** Lakewatch tables can be very large. If the SPL had no time
  clause, that means it relied on the Splunk time picker — translate that with `INRANGE(time)`
  for interactive contexts, or pick a sensible default (7 days) for scheduled rules and surface
  the assumption.
- **Don't translate `lookup`, `inputlookup`, custom search commands, or ES macros silently.**
  These reference Splunk-only storage and behavior; flag them and ask the user where the
  equivalent data lives in Lakewatch.
- **Don't mix `count` and `dc` (distinct count).** SPL `dc(x)` → SQL `COUNT(DISTINCT x)`, not
  `COUNT(x)`. SPL `count` → SQL `COUNT(*)`.
- **Don't backquote OCSF struct paths.** `user.name` is correct; `` `user.name` `` makes it a
  flat column lookup that will fail.
- **Don't treat Splunk multi-value fields as scalars.** If `user` is multi-valued in SPL,
  translate via `EXPLODE` or `array_contains`, not `=`.

## Reference files

- `references/spl-operators.md` — SPL command → Spark SQL translations with examples.
- `references/sourcetype-mapping.md` — Splunk sourcetypes → Lakewatch presets + column mappings.
- `references/cim-to-ocsf.md` — Splunk CIM datamodels → OCSF gold class + field mappings.
- `references/ocsf-gold-tables.md` — Concise list of Lakewatch OCSF gold tables and their
  `class_uid`. **Symlinked** from the `lakewatch-kql-migration` skill; edit in one place.
