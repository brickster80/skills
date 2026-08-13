# Sumo Logic operator → Databricks SQL translation

This is the operator-level cheat sheet. Read the rows that apply to the query in front of you;
don't memorize the whole file.

**The single biggest idea**: in Sumo you `parse` / `json` / `keyvalue` fields *out of* `_raw`
before you can use them. On Lakewatch the preset already did that extraction at ingest — the field
is a typed OCSF struct path. So **most Sumo extraction operators translate to "nothing" — you just
reference the promoted column.** Only translate an extraction step when the field is *not* promoted
(it lives in `unmapped` VARIANT, or you dropped to a silver `data` VARIANT column).

## Table of contents

1. [Query scope → WHERE](#query-scope--where)
2. [Aggregations (`count`, `sum`, `values`, …)](#aggregations)
3. [`count_frequent`, `top`, `topk`](#count_frequent-top-topk)
4. [Time bucketing: `timeslice` / `_timeslice`](#time-bucketing-timeslice--_timeslice)
5. [Filtering and projection (`where`, `if`, `fields`, `dedup`)](#filtering-and-projection)
6. [Field extraction (`parse`, `parse regex`, `json`, `keyvalue`, `csv`, `split`)](#field-extraction)
7. [Window / running aggregates (`accum`, `backshift`, `outlier`, …)](#window--running-aggregates)
8. [Sessions and transactions (`transaction`, `sessionize`)](#sessions-and-transactions)
9. [Joins and lookups (`join`, `lookup`, `save`, `cat`)](#joins-and-lookups)
10. [Multivalue and `... multi`](#multivalue-and-multi)
11. [Strings, parsing, regex](#strings-parsing-regex)
12. [Conditionals and type casts](#conditionals-and-type-casts)
13. [Time, dates, and intervals](#time-dates-and-intervals)
14. [Geo / IP operators](#geo--ip-operators)
15. [Charting (`transpose`)](#charting)
16. [Sumo-only constructs to flag](#sumo-only-constructs-to-flag)

---

## Query scope → WHERE

Everything before the first `|` selects raw messages. It's a mix of metadata filters and keyword
terms combined with `AND`/`OR`/`NOT` (space = implicit AND). It all becomes `WHERE` predicates.

| Sumo scope | Databricks SQL | Notes |
| --- | --- | --- |
| `_sourceCategory=aws/cloudtrail` | `WHERE metadata.log_name = 'cloudtrail'` | **No `_sourceCategory` column exists.** Map to the preset's `metadata.log_name`. See `sourcecategory-mapping.md`. |
| `_sourceCategory=prod/*/nginx` | `WHERE metadata.log_name = 'nginx'` (or the matching preset) | Sumo `*` wildcards in the category are collection-side; pick the Lakewatch preset that ingests that data. |
| `_sourceHost=web-01` | `WHERE src_endpoint.hostname = 'web-01'` (or `device.hostname`) | Role-dependent — see metadata table in `sourcecategory-mapping.md`. |
| `_index=sumologic_audit_events` | (no direct equivalent) | Sumo Partition / Scheduled View. Ask the user which Lakewatch table holds this; flag it. |
| bare keyword `error` | `WHERE <text_col> ILIKE '%error%'` | Full-text over `_raw`. On gold there's rarely a raw text column; usually you filter on a promoted status/severity field instead. Flag if the keyword was load-bearing. |
| `foo error 404` (space=AND) | `WHERE … AND … AND …` | |
| `(error OR fail)` | `WHERE (… OR …)` | |
| `NOT healthcheck` / `!healthcheck` | `WHERE NOT (…)` | |
| `"login failed"` (quoted phrase) | `WHERE <col> ILIKE '%login failed%'` | Literal phrase match. |
| `field=value` (post-scope) | `WHERE field = 'value'` | Sumo comparisons in scope are case-**insensitive** for keywords, case-**sensitive** for field values. |

**Case sensitivity gotcha**: Sumo keyword search is case-insensitive, but `where x = "Foo"`,
`matches`, and regex are case-sensitive. Match that: use `=`/`regexp_like` for case-sensitive,
`ILIKE`/`LOWER(...)` for case-insensitive.

## Aggregations

General Sumo shape: `funcName(field) [as alias] [by groupFields]`. No `as` → auto-named `_count`,
`_sum`, `_avg`, etc. `by` is `GROUP BY`; `group` is a synonym for `by`.

| Sumo | Databricks SQL |
| --- | --- |
| `count` | `SELECT COUNT(*) FROM t` |
| `count by user` | `SELECT user, COUNT(*) AS _count FROM t GROUP BY user` |
| `count as hits by user` | `SELECT user, COUNT(*) AS hits FROM t GROUP BY user` |
| `count(field)` | `SELECT COUNT(field) FROM t` (counts non-null) |
| `count_distinct(ip) by user` | `SELECT user, COUNT(DISTINCT ip) FROM t GROUP BY user` |
| `sum(bytes) by host` | `SELECT host, SUM(bytes) FROM t GROUP BY host` |
| `avg(rt) as art by svc` | `SELECT svc, AVG(rt) AS art FROM t GROUP BY svc` |
| `min(x) as lo, max(x) as hi` | `SELECT MIN(x) AS lo, MAX(x) AS hi FROM t` |
| `stddev(cpu) by inst` | `SELECT inst, STDDEV(cpu) FROM t GROUP BY inst` |
| `pct(rt, 95) as p95` | `SELECT PERCENTILE_APPROX(rt, 0.95) AS p95 FROM t` (exact: `PERCENTILE(rt, 0.95)`) |
| `pct(rt, 75, 95)` | two `PERCENTILE_APPROX` calls, one per percentile |
| `values(ip) as ips by user` | `SELECT user, COLLECT_SET(ip) AS ips FROM t GROUP BY user` |
| `first(ts) as f, last(ts) as l by u` | `SELECT u, MIN(ts) AS f, MAX(ts) AS l …` **only if sorted by time**; otherwise `FIRST_VALUE`/`LAST_VALUE OVER (...)`. Sumo `first`/`last` are sort-order dependent. |
| `most_recent(x_withtime) by user` | `QUALIFY ROW_NUMBER() OVER (PARTITION BY user ORDER BY time DESC) = 1` then select `x` — Sumo `most_recent` picks the value with the newest timestamp |
| `least_recent(x_withtime) by user` | same but `ORDER BY time ASC` |

**`values` caveat**: Sumo `values(x)` returns up to the first **100 distinct** values,
lexicographically sorted. `COLLECT_SET(x)` returns *all* distinct values, unordered. If the
100-cap or ordering matters: `SLICE(ARRAY_SORT(COLLECT_SET(x)), 1, 100)`.

**Most-recent-row per group** (Sumo idiom via `most_recent`/`withtime`, or `first`/`last` after a
`sort`):

```sql
SELECT * FROM t
QUALIFY ROW_NUMBER() OVER (PARTITION BY user ORDER BY time DESC) = 1
```

## `count_frequent`, `top`, `topk`

| Sumo | Databricks SQL |
| --- | --- |
| `count_frequent by user` | `SELECT user, COUNT(*) AS n FROM t GROUP BY user ORDER BY n DESC` (Sumo's is an approximate streaming top-values op; the exact SQL form is fine for translation) |
| `count_frequent by user, action` | `SELECT user, action, COUNT(*) AS n FROM t GROUP BY user, action ORDER BY n DESC` |
| `count by user \| top 10 user by _count` | `… GROUP BY user ORDER BY COUNT(*) DESC LIMIT 10` |
| `top 5 user by _count asc` | `… ORDER BY COUNT(*) ASC LIMIT 5` (rare / least-frequent) |
| `topk(10, field) by grp` | `QUALIFY ROW_NUMBER() OVER (PARTITION BY grp ORDER BY <metric> DESC) <= 10` |

`count_frequent` outputs `_count` and `_approxcount`; if the user relied on the approximate
behavior at huge cardinality, note that Spark's `COUNT(*)` is exact (slower but correct).

## Time bucketing: `timeslice` / `_timeslice`

`timeslice` buckets messages into fixed-width (or fixed-count) intervals and creates `_timeslice`
(bucket start). It must be followed by an aggregation grouping on `_timeslice`.

| Sumo | Databricks SQL |
| --- | --- |
| `timeslice 1h \| count by _timeslice` | `SELECT date_trunc('HOUR', time) AS _timeslice, COUNT(*) FROM t GROUP BY 1 ORDER BY 1` |
| `timeslice 5m \| count by _timeslice` | `SELECT from_unixtime(unix_timestamp(time) - unix_timestamp(time) % 300) AS _timeslice, COUNT(*) …` (Spark has no native 5-min bin) |
| `timeslice 1d \| count by _timeslice, host` | `SELECT date_trunc('DAY', time) AS _timeslice, host, COUNT(*) FROM t GROUP BY 1, 2` |
| `timeslice 150 buckets \| count by _timeslice` | fixed *number* of buckets across the range — compute bucket width from the selected range, or (simpler) pick a fixed unit and note the difference |
| `timeslice by 1m` (older form) | same as `timeslice 1m` |

Units: `w` weeks, `d` days, `h` hours, `m` minutes, `s` seconds.

## Filtering and projection

| Sumo | Databricks SQL | Notes |
| --- | --- | --- |
| `where status_code = 500` | `WHERE status_code = 500` | |
| `where user <> "root"` | `WHERE user <> 'root'` | |
| `where x >= 10 and x <= 20` | `WHERE x BETWEEN 10 AND 20` | |
| `where x in ("a","b")` | `WHERE x IN ('a','b')` | |
| `where x not in (3,4,5)` | `WHERE x NOT IN (3,4,5)` | |
| `where isNull(cc)` | `WHERE cc IS NULL` | |
| `where isEmpty(s)` | `WHERE s = ''` (or `s IS NULL OR s = ''`) | Sumo `isEmpty` = empty string |
| `where isBlank(s)` | `WHERE s IS NULL OR trim(s) = ''` | Sumo `isBlank` = null or whitespace-only |
| `where x matches "fail*"` | `WHERE x ILIKE 'fail%'` | Sumo `matches` with `*` wildcard |
| `where x matches /regex/` | `WHERE regexp_like(x, 'regex')` | Sumo `matches` with `/…/` = regex |
| `where !(x matches /re/)` | `WHERE NOT regexp_like(x, 're')` | |
| `fields user, status, bytes` | `SELECT user, status, bytes` | |
| `fields -_raw, -_messageid` | list the columns you *do* want | Spark has no "all but" projection |
| `dedup by country` | `SELECT DISTINCT …` or `QUALIFY ROW_NUMBER() OVER (PARTITION BY country ORDER BY time DESC) = 1` | sort before dedup to pick the survivor |
| `dedup 3 by src` | `QUALIFY ROW_NUMBER() OVER (PARTITION BY src ORDER BY time) <= 3` | keep first 3 per src |
| `dedup consecutive by src` | needs `LAG(...)` comparison — only drops back-to-back dupes | rare; flag if used |
| `sort by _count` | `ORDER BY _count DESC` | **Sumo `sort` defaults to DESCENDING** |
| `sort by +field` / `sort by field asc` | `ORDER BY field ASC` | |
| `limit 100` | `LIMIT 100` | |

## Field extraction

**Read this section's intro first**: on Lakewatch these usually **disappear**. Check
`sourcecategory-mapping.md` — if the preset promotes the field, drop the extraction and use the
OCSF struct path. Only translate when the field is in `unmapped` or a silver `data` column.

| Sumo | If the field is NOT promoted (translate it) | Notes |
| --- | --- | --- |
| `parse "user=* ip=*" as user, ip` | `regexp_extract(x, 'user=(\\S+) ip=(\\S+)', 1) AS user, …` | Sumo `*` is non-greedy; anchor on the literals around it |
| `parse "user=*" as user nodrop` | same, but don't add a filter that drops non-matches | `nodrop` = keep rows that didn't match |
| `parse regex "(?<ip>\d+\.\d+\.\d+\.\d+)"` | `regexp_extract(x, '(\\d+\\.\\d+\\.\\d+\\.\\d+)', 1) AS ip` | RE2 named groups → indexed `regexp_extract` |
| `parse regex field=path "(?<a>.*?)/"` | `regexp_extract(path, '(.*?)/', 1) AS a` | `field=` sets the input column |
| `json "userIdentity.userName" as u` | `variant_get(data, '$.userIdentity.userName', 'string') AS u` (silver) — **or just `actor.user.name`** if promoted | JSONPath → `variant_get` on a VARIANT column |
| `json "arr[*].type" as t` | `variant_get(data, '$.arr', 'array<variant>')` + `EXPLODE` | array-wildcard = multivalue |
| `json auto` | can't blanket-translate — pick the specific fields the query later uses | `json auto` extracts all keys |
| `keyvalue "k1","k2"` / `kv auto` | `variant_get` per key if in VARIANT, else the promoted column | key=value extraction |
| `csv _raw extract 1 as a, 2 as b` | `split(x, ',')[0] AS a, split(x, ',')[1] AS b` | 1-indexed in Sumo, 0-indexed in Spark |
| `split _raw delim=':' extract 1 as a` | `split(x, ':')[0] AS a` | |

## Window / running aggregates

| Sumo | Databricks SQL |
| --- | --- |
| `accum count as running` | `SUM(count) OVER (ORDER BY time ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running` |
| `accum count as running by user` | add `PARTITION BY user` to the OVER clause (resets per user) |
| `backshift field 1 as prev` | `LAG(field, 1) OVER (ORDER BY time) AS prev` |
| `diff field as d` | `field - LAG(field) OVER (ORDER BY time) AS d` |
| `smooth field 5 as sm` | `AVG(field) OVER (ORDER BY time ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS sm` |
| `rollingstd field 5 as rs` | `STDDEV(field) OVER (ORDER BY time ROWS BETWEEN 4 PRECEDING AND CURRENT ROW) AS rs` |
| `outlier field` | no one-liner — compute a moving mean + stddev window and flag points outside N·σ. Surface as a multi-line CTE and note the band width. |
| `predict field` | forecasting — no SQL equivalent; flag as a separate workflow (Databricks has `ai_forecast` / MLflow). |

## Sessions and transactions

`transaction` / `transactionize` / `sessionize` group related events by a shared key (+ time
window). No single-line SQL equivalent — it's a CTE + window-function pattern, same as SPL
`transaction`.

```sql
-- Sumo: ... | transaction on session_id  (or sessionize by session_id)
WITH sessionized AS (
  SELECT *,
    SUM(CASE
      WHEN unix_timestamp(time) - LAG(unix_timestamp(time))
           OVER (PARTITION BY session_id ORDER BY time) > 30 * 60
      THEN 1 ELSE 0 END
    ) OVER (PARTITION BY session_id ORDER BY time) AS session_seq
  FROM t
)
SELECT session_id, session_seq,
       MIN(time) AS session_start, MAX(time) AS session_end,
       COUNT(*) AS event_count
FROM sessionized
GROUP BY session_id, session_seq
```

If the Sumo `transaction` defined explicit start/end **states**, add a CASE that detects the
marker event and resets `session_seq`. Surface the maxspan / state assumptions in mapping notes.

## Joins and lookups

### `join`

Sumo `join` is an inner join of subqueries with an `on` clause (AND conditions only), optional
`timewindow`. After the join, fields are referenced `t1_field` / `t2_field`.

```
... | join (parse "a=*" as a) as t1, (parse "b=*" as b) as t2 on t1.a = t2.b
```

```sql
SELECT t1.*, t2.b
FROM (SELECT … AS a FROM events) t1
INNER JOIN (SELECT … AS b FROM events) t2 ON t1.a = t2.b
```

Sumo `join` has hard input limits (~25k msgs/table). If the source query relied on that limit for
correctness, note it. `timewindow 5m` → add `AND ABS(unix_timestamp(t1.time) -
unix_timestamp(t2.time)) <= 300` to the `ON`.

### `lookup`

Sumo `lookup` enriches from a saved lookup table:

```
... | lookup dept, mgr from path://"/Library/.../Users" on userEmail=email
```

```sql
SELECT t.*, l.dept, l.mgr
FROM events t
LEFT JOIN <catalog>.<schema>.users_lookup l ON l.userEmail = t.email
```

The lookup table has to exist in Lakewatch as a Delta table. **Ask the user where the lookup data
lives** — don't silently drop the enrichment. Built-in `lookup … from geo://location on ip=IP` →
see [Geo / IP](#geo--ip-operators).

### `save` / `cat`

`save path://"…"` writes results into a Sumo lookup table; `cat path://"…"` reads one. Both are
Sumo-only storage. Flag them and ask the user for the Lakewatch equivalent (a Delta table write,
or an existing table to read).

## Multivalue and `... multi`

| Sumo | Databricks SQL |
| --- | --- |
| `parse regex "(?<ip>…)" multi` | `LATERAL VIEW EXPLODE(regexp_extract_all(x, '…', 1)) AS ip` — one row per match |
| `json "arr[*].field" as f` | `EXPLODE(variant_get(data, '$.arr', 'array<variant>'))` then project `.field` |
| `values(x) by g` | `COLLECT_SET(x)` (see aggregations) |
| `jsonArrayContains(f, "x")` | `array_contains(f, 'x')` |
| `jsonArraySize(f)` | `SIZE(f)` |

## Strings, parsing, regex

| Sumo | Databricks SQL |
| --- | --- |
| `length(s)` | `LENGTH(s)` |
| `toUpperCase(s)` / `toLowerCase(s)` | `UPPER(s)` / `LOWER(s)` |
| `substring(s, 1, 3)` | `SUBSTRING(s, 2, 2)` — **Sumo `substring` is 0-indexed with (start, end); Spark `SUBSTRING` is 1-indexed with (start, len). Convert carefully.** |
| `concat(a, "-", b)` | `CONCAT(a, '-', b)` |
| `replace(s, "old", "new")` | `REPLACE(s, 'old', 'new')` (literal) / `REGEXP_REPLACE(s, 're', 'new')` (regex) |
| `format("%s:%d", a, b)` | `FORMAT_STRING('%s:%d', a, b)` |
| `x matches "pat*"` | `regexp_like(x, '^pat.*$')` or `x ILIKE 'pat%'` |
| `x matches /re/` | `regexp_like(x, 're')` |
| `parse regex ... nodrop` | omit the implicit match-required filter |

**Regex gotcha**: Sumo uses **RE2** (Go's regex), Spark uses **Java regex**. Most patterns port
cleanly, but RE2 has no backreferences or lookaround — if a Sumo pattern used those it wasn't RE2
to begin with (probably a `parse` anchor), so re-check. Named groups `(?<name>…)` are referenced by
index in `regexp_extract`, not by name.

### Command-line and switch matching

Queries that search command lines / EDR process data for tool names and CLI switches are the most
common place naive translations fail.

**Tool / executable names** (e.g. `powershell`). A loose `ILIKE '%powershell%'` also matches
`mypowershellscript.txt`. Safer:

```sql
regexp_like(LOWER(process.cmd_line), '\\bpowershell')
```

**CLI switches** (e.g. `-enc`). A naive `ILIKE '%-enc%'` matches `-encrypted`, `-enchant`. Anchor
on explicit delimiters:

```sql
regexp_like(LOWER(process.cmd_line), '(^|[\\s"''])[-/]e(nc(odedcommand)?|c)(\\b|$)')
-- matches: -enc, -Enc, -encodedcommand, -ec, /EncodedCommand ; NOT -encrypted, -enchant
```

`\b` alone is too clever for shell tokens that begin with `-` or `/`.

## Conditionals and type casts

| Sumo | Databricks SQL |
| --- | --- |
| `if(cond, a, b) as x` | `IF(cond, a, b) AS x` (or `CASE WHEN cond THEN a ELSE b END`) |
| `cond ? a : b as x` (ternary) | `IF(cond, a, b) AS x` |
| nested `if(...)` | nested `CASE WHEN … WHEN … ELSE … END` |
| `toLong(s) as n` | `CAST(s AS BIGINT) AS n` |
| `num(s) as n` | `CAST(s AS DOUBLE) AS n` |
| `isNull(x)` | `x IS NULL` |
| `isEmpty(x)` | `x = ''` |
| `isBlank(x)` | `x IS NULL OR trim(x) = ''` |
| `isNumeric(x)` | `x RLIKE '^-?[0-9]+(\\.[0-9]+)?$'` |
| `a - b as delay` (arithmetic) | `a - b AS delay` — but if `a`,`b` are `_receiptTime`/`_messageTime` (epoch ms), use `unix_millis(...)` differences or `timestampdiff` |

## Time, dates, and intervals

Sumo's time range comes from the **UI time picker or the Search Job API `from`/`to`** — almost
never from the query body. This is even more consistently true than for Splunk. So the default
translation of "the query's implicit time window" is `INRANGE(time)`.

| Sumo | Databricks SQL |
| --- | --- |
| *(no time clause in the query — the norm)* | `INRANGE(time)` for interactive; `time >= current_timestamp() - INTERVAL <n> DAY` for scheduled rules |
| `now()` | `current_timestamp()` (Sumo `now()` is epoch **ms**; Spark returns a timestamp) |
| `queryStartTime()` | the picker's start — `INRANGE(time)` covers this on the interactive side |
| `queryEndTime()` | the picker's end — same |
| `_messageTime` | `time` (canonical event timestamp on every Lakewatch table) |
| `_receiptTime` | `metadata.logged_time` or `metadata.processed_time` (ingest time) |
| `_receiptTime - _messageTime as delay` | `unix_millis(metadata.processed_time) - unix_millis(time) AS delay_ms` (or `timestampdiff(SECOND, time, metadata.processed_time)`) |
| `timeslice 1h` | `date_trunc('HOUR', time)` — see [time bucketing](#time-bucketing-timeslice--_timeslice) |
| `formatDate(_messageTime, "yyyy-MM-dd")` | `date_format(time, 'yyyy-MM-dd')` |
| `parseDate(s, "MM/dd/yyyy")` | `to_timestamp(s, 'MM/dd/yyyy')` |
| `formatDate(_messageTime, "EEE") \| where day="Mon"` | `date_format(time, 'EEE') = 'Mon'` |

### `INRANGE(time)` — bind to the UI timespan selector

Lakewatch's interactive SQL surfaces (explore view, dashboard widgets) expose a **timespan
selector**. `INRANGE(time)` resolves at runtime to the selected range.

**Default for Sumo: use `INRANGE(time)`.** A Sumo search is, in real-world use, run against
whatever range the picker has selected — the query text has no `earliest=`/`latest=` equivalent at
all. `INRANGE(time)` reproduces exactly that UX on Lakewatch.

**Use a literal `INTERVAL` only when:**

- The user says they're wrapping the result into a **scheduled rule** (rules get no picker; the
  lookback must be concrete and tied to `schedule.atLeastEvery`).
- The user explicitly wants a hard pin.
- The Sumo source is a **Scheduled Search** with a fixed range in its schedule config (that range
  was deliberate — use it).

**Preserve in-body time logic literally.** If the query uses `_messageTime` / `_receiptTime` /
`queryStartTime()` arithmetic (e.g. `where _messageTime > queryStartTime() + 3600000`), that was a
deliberate in-pipeline time computation — translate it faithfully rather than replacing it with
`INRANGE`.

## Geo / IP operators

| Sumo | Databricks SQL |
| --- | --- |
| `geoip remote_ip` (adds `country_code`, `city`, `latitude`, …) | Lakewatch presets often already compute `src_endpoint.location.country` / `.city` at ingest — use those. If not present, geo enrichment is a separate dataset; flag it. |
| `lookup lat, long, cc from geo://location on ip=IP` | same as above — prefer the promoted `*.location.*` OCSF paths |
| `ipv4ToNumber(ip)` | no built-in; cast octets, or flag if used for range compares |
| `isPublicIP(ip)` / `isPrivateIP(ip)` | `NOT (ip LIKE '10.%' OR ip LIKE '192.168.%' OR ip RLIKE '^172\\.(1[6-9]|2[0-9]|3[01])\\.')` etc. — write it out or flag |
| `CIDR` / `isValidIP` | write the predicate explicitly or flag |

## Charting

| Sumo | Databricks SQL |
| --- | --- |
| `timeslice 1h \| count by _timeslice` | `SELECT date_trunc('HOUR', time) AS _timeslice, COUNT(*) … GROUP BY 1 ORDER BY 1` — feeds a Lakewatch dashboard line/column widget directly |
| `timeslice 1h \| count by _timeslice, status \| transpose row _timeslice column status` | Lakewatch chart widgets pivot on the GROUP BY columns automatically — **don't** manually `PIVOT` unless a wide output is genuinely required. Keep the long form `GROUP BY _timeslice, status`. |

`transpose` reshapes rows → columns (pivot). Only translate to SQL `PIVOT` if the downstream
consumer truly needs a wide table; for dashboard widgets the long form is what Lakewatch wants.

## Sumo-only constructs to flag

These have no clean Lakewatch equivalent. Surface them in mapping notes and ask the user how to
resolve:

- `_index=<partition>` / `_view=<scheduled_view>` — Sumo Partitions / Scheduled Views are
  Sumo-side storage/acceleration. Ask which Lakewatch table holds the equivalent data.
- `lookup … from path://"…"` / `save` / `cat` — Sumo lookup tables. Need a Delta table in
  Lakewatch; ask the user.
- `outlier` / `predict` — Sumo's built-in anomaly/forecast ops. No SQL one-liner; point the user
  at Databricks `ai_forecast` / anomaly detection as a separate workflow.
- `threatip` / threat-intel enrichment — Sumo-hosted intel feed. Ask where the equivalent IOC
  data lives in Lakewatch.
- `logcompare` / `logreduce` / `logexplain` — Sumo ML pattern operators. No SQL equivalent; flag
  as a separate analysis (Lakewatch can use `ai_classify` for similar classification).
- `merge` / `trace` — transaction/session helpers; flag and translate the underlying grouping if
  feasible.
