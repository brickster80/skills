---
name: lakewatch-kql-migration
description: >
  Translate Kusto Query Language (KQL) into Databricks SQL for Databricks Lakewatch. Use this skill
  whenever the user asks to migrate, port, rewrite, convert, or translate KQL into SQL — including
  any mention of Microsoft Sentinel queries, Defender Advanced Hunting / 365 Defender / Defender XDR
  queries, Azure Data Explorer, KQL operators (`summarize`, `extend`, `project`, `mv-expand`,
  `make-series`, `bin()`, `ago()`, `arg_max`, `parse`, etc.), Sentinel tables (SecurityEvent,
  SigninLogs, AzureActivity, …), Defender Advanced Hunting tables (DeviceProcessEvents,
  DeviceFileEvents, EmailEvents, …), or moving detections from Sentinel/Defender into Lakewatch.
  Use it even if the user does not say the word "skill" or "Lakewatch" — if there is KQL involved
  and the destination is Databricks, this skill applies. The translation produces Databricks SQL
  plus inline mapping notes (chosen target table, operator/field mapping, caveats).
---

# Lakewatch KQL → SQL migration

This skill translates Kusto Query Language (KQL) queries — typically from **Microsoft Sentinel**
or **Microsoft Defender XDR (Advanced Hunting)** — into Databricks SQL that runs on
**Databricks Lakewatch**.

The goal is a translation that:

1. Runs unmodified against Lakewatch tables.
2. Uses the **correct Lakewatch destination table** (OCSF gold tables first; silver vendor tables
   only when the gold table cannot represent the data).
3. Comes with **inline mapping notes** so a security engineer can review the choice of table,
   operator translation, and any semantic caveats.

## Why this is harder than a textbook SQL conversion

KQL and Databricks SQL look superficially similar but differ in three big ways that drive most of
the translation work:

- **Schema is not a 1:1 rename.** A Sentinel `SigninLogs` row or a Defender `DeviceProcessEvents`
  row does not exist as a table in Lakewatch. Lakewatch normalizes events to a **modified OCSF
  1.3.0** gold layer (`authentication`, `process_activity`, `network_activity`, …). One source
  table maps to one OCSF class; many source columns get renamed, restructured into nested structs,
  or fold into enums (e.g., `ResultType=0` → `status='Success'`).
- **Operators don't all map cleanly.** `summarize arg_max(timestamp, *) by user`, `mv-expand`,
  `make-series`, `parse`, `bin()`, and `ago()` need careful Spark SQL equivalents — sometimes
  with `QUALIFY` + window functions, `explode`, `regexp_extract`, `date_trunc`, or `INTERVAL`.
- **Lookback windows shift meaning.** KQL queries often rely on the ambient Sentinel/Defender
  90-day retention. Lakewatch rules schedule themselves; the lookback window has to match the
  rule's `atLeastEvery` (see [[lakewatch-rule-dev]] if you are building a full detection rule).

Read the WHY before forcing a translation — if the source KQL is doing something that depends on
Sentinel-only behavior (e.g., `_GetWatchlist()`, `externaldata()`, Sentinel functions), surface
that in the mapping notes instead of guessing.

## Workflow

### 1. Read the KQL carefully

Before writing any SQL, identify:

- **Source tables** referenced in the `FROM`-equivalent position (the table name at the very start,
  or after `union` / `join`).
- **Time filter** — is it `ago(7d)`, `between(...)`, or `TimeGenerated > datetime(...)`?
- **Aggregations** — what is being `summarize`'d, grouped, or windowed.
- **Output shape** — what columns the final `project` produces (this drives output template if the
  user is building a detection rule).

If any source table is unfamiliar, **do not guess its purpose** — ask the user or look it up.
Sentinel and Defender tables overlap in confusing ways (e.g., `SecurityEvent` ≠
`DeviceProcessEvents`).

### 2. Pick the destination table

The decision tree:

1. **Is there a matching OCSF gold table?** This is the strongly preferred target — rules, joins,
   and dashboards built on gold are vendor-neutral.
   - Sentinel `SigninLogs` → `authentication`
   - Defender `DeviceProcessEvents` → `process_activity`
   - See `references/sentinel-table-mapping.md` and `references/defender-table-mapping.md` for the
     full list. **Read the relevant reference file before picking** — guessing leads to wrong
     `class_uid` filtering.
2. **If no gold mapping exists**, fall back to the **silver vendor table** (e.g., `mde_device_info`,
   `mde_identity_info`, `mde_email_url_info`). Silver tables keep the raw `data` VARIANT column —
   you'll use `try_variant_get(data, '$.properties.<FieldName>', '<type>')` to reach fields.
3. **If even silver doesn't exist**, tell the user the source isn't ingested by Lakewatch and stop.
   Don't invent a target table.

When uncertain, verify against the live catalog rather than guessing — use the `databricks-dbsql`
or `databricks-unity-catalog` skill to `DESCRIBE` the candidate table before committing to the
translation. Schemas evolve and the bundled references can lag.

**Source filter rule of thumb**: gold tables hold events from multiple presets, so almost every
translated query needs a source filter in its `WHERE`. Prefer **`metadata.log_name`** (the
preset's `sourceType` — short string like `defender_xdr`, `microsoft_entra_authentication`,
`azure_activity_logs`) as the primary filter. It's smaller on disk than `metadata.product.name`,
gets better Delta data-skipping, and is the most stable value across upgrades. Add
`metadata.product.name` only when `log_name` is too coarse for the discrimination you need
(common example: on `authentication`, both Defender for Endpoint's `DeviceLogonEvents` and
Defender for Identity's `IdentityLogonEvents` carry `log_name = 'defender_xdr'`, so you need
`product.name` to pick one).

### 3. Translate KQL operators

`references/kql-operators.md` is the operator-level cheat sheet. Read it whenever the source query
uses anything beyond `where` / `project` / `summarize count() by …`.

Patterns worth knowing up front:

- `ago(7d)` → `time >= current_timestamp() - INTERVAL 7 DAY`
- **No time filter at all in the KQL** → `INRANGE(time)` (Lakewatch's bind-to-UI-timespan-picker
  primitive). The Sentinel UI supplies the lookback; reproduce that on the Lakewatch side. Only
  use `INRANGE(time)` for **interactive** SQL (explore / dashboard) — for detection rules, pick a
  sensible interval and surface it in the mapping notes. See `references/kql-operators.md` →
  *Time, dates, and intervals* → *`INRANGE(time)`*.
- `summarize arg_max(time, *) by user` → `QUALIFY ROW_NUMBER() OVER (PARTITION BY user ORDER BY time DESC) = 1`
- `mv-expand col` → `LATERAL VIEW EXPLODE(col)` (or `EXPLODE` in a `SELECT`)
- `bin(time, 1h)` → `date_trunc('HOUR', time)`
- `parse … with` → `regexp_extract(..., '...', N)` (multiple captures need multiple calls)
- `extend foo = …` → just put the expression in the `SELECT`. Don't reach for CTEs unless the
  derived value is used multiple times or feeds an aggregate.
- `has` on command lines (e.g., `ProcessCommandLine has 'powershell'` or `has '-enc'`) → a
  textbook `\bword\b` regex is **wrong** for tool names and CLI switches. Read
  `references/kql-operators.md` → *Strings, parsing, regex* → *Command-line and switch matching*
  before translating; in short: anchor on explicit delimiters at the start, be permissive on the
  right (e.g., `-enc` should also catch `-EncodedCommand`).

Don't introduce a CTE per `extend` step — the resulting SQL becomes unreadable. Inline expressions
in the `SELECT` are easier to review and Spark optimizes them identically.

### 4. Map field references

After translating operators, map every column reference. The biggest gotchas:

- **The preset is the contract, not the OCSF spec.** Each Lakewatch preset chooses which source
  fields to promote to typed OCSF struct paths and which to keep in the `unmapped` VARIANT
  column. **Read the relevant `references/<vendor>-table-mapping.md` row before naming a target
  path** — guessing from OCSF docs alone produces SQL that returns NULL for the field you thought
  you mapped. Common gotchas in the Defender preset alone: `Subject` is in `message` (not
  `email.subject`), `SenderFromAddress` is in `unmapped` (not `email.from`), `Url` is in
  `http_request.url` (not `url.url_string`), `Workload` is in `unmapped` (there is no
  `metadata.product.feature.name`), `ThreatTypes` is a delimited string in `unmapped` (not a
  `threat.type` array). Network events on `network_activity` keep initiating-process fields in
  `unmapped` rather than `actor.process.*`.
- **OCSF gold puts mapped fields in nested structs.** `AccountUpn` becomes `user.name` (on
  `authentication`). `IPAddress` becomes `src_endpoint.ip`. `ResultType=0` becomes
  `status='Success'` (and `status_id=1`).
- **Defender source fields live under `data:$.properties.<Name>`** when you're on a silver table.
  Use `try_variant_get(data, '$.properties.ProcessCommandLine', 'string')`.
- **`time` is the canonical timestamp column** in every Lakewatch silver and gold table. Don't
  preserve Sentinel's `TimeGenerated` / `Timestamp`.

Field-level mappings for the two largest sources are in:
- `references/sentinel-table-mapping.md` — Sentinel tables and their column → OCSF mappings.
- `references/defender-table-mapping.md` — Defender Advanced Hunting tables and their column →
  OCSF (or silver `data` path) mappings.

### 5. Write the output

Produce **two parts**:

**Part A — the translated SQL.** Runnable as-is. Use lowercase keywords or uppercase consistently
(match the user's existing style if they have one in scope). Place a `-- ` header comment at the top
naming the destination table and the original source.

**Part B — mapping notes.** Below the SQL, in a fenced block titled `Mapping notes`, include:

- The **destination table** chosen and **why** (which OCSF class it maps to).
- A short **column mapping table** showing source KQL column → SQL expression. Only include rows
  where the mapping is non-obvious (renames, struct lookups, type casts, enum collapses).
- Any **semantic caveats** — fields that don't have a clean equivalent, lookback assumptions, joins
  that had to be restructured.
- If you fell back to a silver table or had to skip a piece of the query, **say so explicitly** —
  reviewers should not have to diff the queries to discover it.

### Example output shape

```sql
-- Source: Sentinel SigninLogs — failed sign-ins past 24h grouped by user
-- Destination: <catalog>.gold.authentication (Lakewatch OCSF, fed by the Entra preset)
SELECT
  user.name AS user_name,
  COUNT(*) AS failed_attempts,
  COUNT(DISTINCT src_endpoint.ip) AS distinct_src_ips
FROM gold.authentication
WHERE time >= current_timestamp() - INTERVAL 24 HOUR
  AND metadata.log_name = 'microsoft_entra_authentication'
  AND status != 'Success'
GROUP BY user.name
HAVING COUNT(*) >= 10
```

**Mapping notes**

- **Destination**: `authentication` (OCSF class 3002) — Lakewatch's normalized table for sign-in
  events. Sentinel `SigninLogs` is fed in via the **`microsoft_entra`** preset.
- **Source filter**: `metadata.log_name = 'microsoft_entra_authentication'` reliably isolates Entra
  sign-ins because `authentication` also holds Defender `DeviceLogonEvents`, Windows 4624/4625,
  and others. Note: `metadata.product.name` for Entra is **dynamic** — it can be `'Microsoft Entra
  ID'`, the Sentinel category (`'SigninLogs'`, `'NonInteractiveUserSignInLogs'`, …), or
  `'Azure AD B2C'` depending on the operation. The silver-table name in `metadata.log_name` is
  more stable. If you need a different sub-source (B2C only, non-interactive only), filter by the
  original category value instead.
- **Catalog**: the Lakewatch catalog name **varies by environment** — `<catalog>.gold.authentication`
  rather than a hardcoded `lw-prod-catalog`. Ask the user (or check `SHOW CATALOGS`) which catalog
  their workspace uses; common patterns are `lw-prod-catalog`, `lakewatch_<env>`, or a sandbox
  name. Many Lakewatch SQL contexts run with `gold` already selected as the default schema, so the
  catalog/schema prefix can often be dropped.
- **Column mapping**:

  | Sentinel `SigninLogs` | Lakewatch `authentication` |
  | --- | --- |
  | `UserPrincipalName` | `user.name` |
  | `IPAddress` | `src_endpoint.ip` |
  | `ResultType` (non-zero = fail) | `status != 'Success'` |
  | `TimeGenerated` | `time` |

- **Caveat**: KQL `ResultType` is an integer error code; Lakewatch flattens it to a string status.
  If the original rule needed the exact error code, also project `status_detail` or
  `status_code`.

## When the user wants more than a translation

If the user is **building a detection rule** out of the translated query, route them to the
`lakewatch-rule-dev` skill once translation is done — that skill handles the YAML wrapping, MITRE
mapping, output template, and validator workflow.

If the user is **building a dashboard widget**, point them at `lakewatch-dashboard-dev`.

This skill stops at "here is the SQL + mapping notes." Don't try to also produce a rule YAML in
the same pass — it makes review harder and the rule skill has its own validation steps.

## Common mistakes to avoid

- **Don't invent column names.** If you're unsure whether OCSF `authentication` has `user.uid` or
  `user.id`, check `references/sentinel-table-mapping.md`, the OCSF docs, or `DESCRIBE` the table.
- **Don't strip the time filter.** Lakewatch tables can be very large; a KQL query without `ago()`
  is usually still scoped by Sentinel's retention window. Pick a sensible default (7 days) and
  surface the assumption in the mapping notes.
- **Don't translate `_GetWatchlist()` or `externaldata()` silently.** These reference Sentinel-only
  storage; flag them and ask the user where the equivalent data lives in Lakewatch.
- **Don't mix `count()` and `count_distinct()`.** KQL `dcount(x)` → SQL `COUNT(DISTINCT x)`, not
  `COUNT(x)`.
- **Don't backquote OCSF struct paths.** `user.name` is correct; `\`user.name\`` makes it a
  flat column lookup that will fail.

## Reference files

- `references/kql-operators.md` — KQL operator → Spark SQL translations with examples.
- `references/sentinel-table-mapping.md` — Sentinel tables → Lakewatch tables, column mappings.
- `references/defender-table-mapping.md` — Defender Advanced Hunting tables → Lakewatch tables,
  column mappings.
- `references/ocsf-gold-tables.md` — Concise list of Lakewatch OCSF gold tables and their
  `class_uid`. Use as a quick lookup; the authoritative full schema lives in
  `<catalog>.gold.*` (query via `DESCRIBE`) or in the antimatter content-marketplace repo's
  `gold_tables.md` if available locally.
