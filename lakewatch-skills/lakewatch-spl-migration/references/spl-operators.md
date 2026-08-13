# SPL operator → Databricks SQL translation

This is the operator-level cheat sheet. Read the rows that apply to the query in front of you;
don't memorize the whole file.

## Table of contents

1. [Filtering and projection](#filtering-and-projection)
2. [Aggregations and `stats`](#aggregations-and-stats)
3. [Window-style aggregates: `streamstats`, `eventstats`](#window-style-aggregates-streamstats-eventstats)
4. [`dedup`, `top`, `rare`, `head`, `tail`](#dedup-top-rare-head-tail)
5. [Joins, unions, and lookups](#joins-unions-and-lookups)
6. [`transaction`](#transaction)
7. [`tstats` and accelerated datamodels](#tstats-and-accelerated-datamodels)
8. [Time, dates, and intervals](#time-dates-and-intervals)
9. [Strings, parsing, regex (`rex`, `extract`, `replace`)](#strings-parsing-regex)
10. [Multi-value fields and `mvexpand`](#multi-value-fields-and-mvexpand)
11. [`spath`, `xpath`, JSON paths](#spath-xpath-json-paths)
12. [Conditionals and `eval`](#conditionals-and-eval)
13. [`timechart`, `chart`](#timechart-chart)
14. [Splunk-only constructs to flag](#splunk-only-constructs-to-flag)

---

## Filtering and projection

| SPL | Databricks SQL | Notes |
| --- | --- | --- |
| `search foo=bar` / `where foo="bar"` | `WHERE foo = 'bar'` | SPL `search` and `where` differ in subtle ways but both translate to `WHERE`. Single-quote strings in SQL. |
| `search foo!=bar` | `WHERE foo <> 'bar'` | |
| `search foo=*` | `WHERE foo IS NOT NULL AND foo <> ''` | SPL `*` matches "anything"; SQL needs the explicit not-null check. |
| `search NOT foo=bar` | `WHERE foo <> 'bar'` (or `foo IS NULL OR foo <> 'bar'` if you want NULL-safe) | SPL `NOT` is liberal about NULLs; Spark `<>` returns UNKNOWN, not TRUE. |
| `search foo="*bar*"` | `WHERE foo ILIKE '%bar%'` | Splunk wildcards translate to `ILIKE` (case-insensitive substring). Splunk searches are case-insensitive by default. |
| `search foo IN (a,b,c)` | `WHERE foo IN ('a','b','c')` | |
| `where isnotnull(x)` | `WHERE x IS NOT NULL` | Splunk `isnotnull` also rejects empty strings — use `WHERE x IS NOT NULL AND x <> ''` if you want the same semantics. |
| `fields a b c` | `SELECT a, b, c` | |
| `fields - a` | List the remaining columns explicitly. | Spark has no "all but" projection. |
| `rename old AS new` | `SELECT old AS new` | |
| `eval new = expr` | Inline in `SELECT`: `SELECT expr AS new, …` | Avoid a CTE per `eval` step. |
| `table a b c` | `SELECT a, b, c` (typically at the end) | `table` is roughly `SELECT` with column ordering. |
| `head 100` | `LIMIT 100` | |
| `tail 100` | `ORDER BY time DESC LIMIT 100` | Splunk `tail` reads from the bottom; the SQL equivalent is to sort descending and take the first N. |
| `reverse` | `ORDER BY time DESC` (or whatever sort you want reversed) | |
| `dedup x` | See [dedup](#dedup-top-rare-head-tail) below |  |

## Aggregations and `stats`

| SPL | Databricks SQL |
| --- | --- |
| `stats count` | `SELECT COUNT(*) FROM t` |
| `stats count by user` | `SELECT user, COUNT(*) FROM t GROUP BY user` |
| `stats count AS n by user` | `SELECT user, COUNT(*) AS n FROM t GROUP BY user` |
| `stats dc(ip) by user` | `SELECT user, COUNT(DISTINCT ip) FROM t GROUP BY user` |
| `stats sum(bytes) by ip` | `SELECT ip, SUM(bytes) FROM t GROUP BY ip` |
| `stats min(_time), max(_time) by user` | `SELECT user, MIN(time), MAX(time) FROM t GROUP BY user` |
| `stats avg(latency) by service` | `SELECT service, AVG(latency) FROM t GROUP BY service` |
| `stats p95(latency) by service` | `SELECT service, PERCENTILE_APPROX(latency, 0.95) FROM t GROUP BY service` |
| `stats values(ip) AS ips by user` | `SELECT user, COLLECT_SET(ip) AS ips FROM t GROUP BY user` (distinct values) |
| `stats list(ip) AS ips by user` | `SELECT user, COLLECT_LIST(ip) AS ips FROM t GROUP BY user` (preserves duplicates) |
| `stats earliest(_time), latest(_time) by user` | `SELECT user, MIN(time), MAX(time) FROM t GROUP BY user` (Splunk's earliest/latest are time-ordered min/max) |
| `stats earliest(field) by user` | `SELECT user, FIRST_VALUE(field) OVER (PARTITION BY user ORDER BY time)` — but the natural form needs a CTE; see *most-recent-row* below |
| `stats count(eval(status!="success"))` | `SUM(CASE WHEN status <> 'success' THEN 1 ELSE 0 END)` (or `COUNT_IF(...)` in DBR ≥ 14.0) |
| `stats sum(eval(if(bytes>0,bytes,0)))` | `SUM(CASE WHEN bytes > 0 THEN bytes END)` |

**Most-recent-row per group** (a common SPL pattern that has no single-line `stats` equivalent in
SQL):

```spl
... | stats latest(*) by user
```

```sql
SELECT * FROM t
QUALIFY ROW_NUMBER() OVER (PARTITION BY user ORDER BY time DESC) = 1
```

## Window-style aggregates: `streamstats`, `eventstats`

`streamstats` and `eventstats` are SPL's window-function analogs.

| SPL | Databricks SQL |
| --- | --- |
| `eventstats avg(latency) AS avg_lat by service` | `AVG(latency) OVER (PARTITION BY service) AS avg_lat` |
| `eventstats count by service` | `COUNT(*) OVER (PARTITION BY service)` |
| `streamstats count by user` | `COUNT(*) OVER (PARTITION BY user ORDER BY time ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` (running count) |
| `streamstats window=10 avg(latency)` | `AVG(latency) OVER (ORDER BY time ROWS BETWEEN 9 PRECEDING AND CURRENT ROW)` (sliding window of N rows) |
| `streamstats time_window=1h sum(bytes)` | `SUM(bytes) OVER (ORDER BY time RANGE BETWEEN INTERVAL 1 HOUR PRECEDING AND CURRENT ROW)` (sliding time window) |

**Difference between `streamstats` and `eventstats`**: `eventstats` produces the same aggregate
value on every row in a group (it's a partition-only window); `streamstats` is time-ordered and
running. Match the OVER clause to the SPL form.

## `dedup`, `top`, `rare`, `head`, `tail`

| SPL | Databricks SQL |
| --- | --- |
| `dedup x` | `SELECT DISTINCT x FROM t` (drops duplicates by `x`, keeps only the first row encountered) — for deterministic results, use `QUALIFY ROW_NUMBER() OVER (PARTITION BY x ORDER BY time) = 1` |
| `dedup x sortby -_time` | `QUALIFY ROW_NUMBER() OVER (PARTITION BY x ORDER BY time DESC) = 1` (keep latest row per `x`) |
| `dedup 3 x` | `QUALIFY ROW_NUMBER() OVER (PARTITION BY x ORDER BY time) <= 3` (keep first 3 rows per `x`) |
| `top 10 user` | `SELECT user, COUNT(*) FROM t GROUP BY user ORDER BY COUNT(*) DESC LIMIT 10` |
| `top user by host` | `SELECT host, user, COUNT(*) FROM t GROUP BY host, user QUALIFY ROW_NUMBER() OVER (PARTITION BY host ORDER BY COUNT(*) DESC) = 1` |
| `rare 10 user` | Same as `top` but `ORDER BY COUNT(*) ASC` |
| `head 100` | `LIMIT 100` |

## Joins, unions, and lookups

SPL has three pipeline-join commands (`join`, `lookup`, `append`/`appendcols`) that each have
different semantics. Pay attention to which one the source query uses.

### `join`

SPL `join` is restrictive (left-side rows limited to 50k by default; inner-join semantics with a
hidden subsearch). Translate to a regular SQL join, but **call out the row-limit if the SPL
relied on it**:

```spl
... | join type=inner sessionid [search index=auth | fields sessionid user]
```

```sql
SELECT t1.*, t2.user
FROM t1
INNER JOIN (SELECT sessionid, user FROM auth) t2 ON t1.sessionid = t2.sessionid
```

SPL `join type=left` → `LEFT JOIN`. SPL `join` without `type=` is inner.

### `lookup`

SPL `lookup` is a left-join against a definitions file (CSV, KV store, or external):

```spl
... | lookup user_lookup user OUTPUT department, manager
```

In Lakewatch, the lookup table has to be ingested as a Delta table (e.g., a content-pack
dataset). Ask the user where the lookup data lives, then:

```sql
SELECT t.*, l.department, l.manager
FROM events t
LEFT JOIN <catalog>.<schema>.user_lookup l ON l.user = t.user
```

If the lookup table doesn't exist in Lakewatch, flag it in the mapping notes — don't silently
drop the enrichment.

### `inputlookup`

SPL `inputlookup my.csv` reads a lookup as a result set. Translate to `SELECT … FROM
<lookup_table>`. Same caveat as `lookup`: the table has to exist in Lakewatch.

### `append` / `appendcols`

| SPL | Databricks SQL |
| --- | --- |
| `… | append [search …]` | `UNION ALL` between the two searches |
| `… | appendcols [search …]` | No direct SQL equivalent (column-wise concat without a join key); usually needs to be rewritten as a join, or as two separate queries. Flag this. |

## `transaction`

`transaction` groups events into sessions based on a key + time window + optional start/end
markers. There's no single-line SQL equivalent — it's a CTE + window function pattern.

```spl
... | transaction sessionid maxspan=30m
```

A reasonable translation:

```sql
WITH sessionized AS (
  SELECT
    *,
    -- Group events into sessions: a new session starts when the gap to the previous
    -- event in the same key exceeds 30 minutes.
    SUM(
      CASE
        WHEN UNIX_TIMESTAMP(time) - LAG(UNIX_TIMESTAMP(time))
             OVER (PARTITION BY sessionid ORDER BY time) > 30 * 60
        THEN 1 ELSE 0
      END
    ) OVER (PARTITION BY sessionid ORDER BY time) AS session_seq
  FROM t
)
SELECT
  sessionid,
  session_seq,
  MIN(time) AS session_start,
  MAX(time) AS session_end,
  COUNT(*)  AS event_count
FROM sessionized
GROUP BY sessionid, session_seq
```

SPL `transaction` also produces `duration` and `eventcount` fields. Both are derivable as shown
above. If the SPL uses `startswith=`/`endswith=` to bound sessions, that adds a CASE expression
that detects the marker event and resets `session_seq`.

For dashboards / explore use, often a simpler `MIN(time)`/`MAX(time)`/`COUNT(*) GROUP BY
sessionid` is enough. If the original query relied on `maxspan` to split long sessions, surface
that in mapping notes.

## `tstats` and accelerated datamodels

`tstats` is a fast aggregation against an accelerated datamodel:

```spl
| tstats count from datamodel=Authentication where Authentication.action=failure by Authentication.user
```

Translate by:

1. Mapping the **datamodel** to the matching OCSF gold table (see
   `references/cim-to-ocsf.md`). `Authentication` → `authentication`.
2. Stripping the datamodel prefix from field names (`Authentication.user` → `user`, then the
   CIM→OCSF rename `user` → `user.name`).
3. Running a regular `SELECT … COUNT(*) … GROUP BY` against the gold table.

```sql
SELECT user.name, COUNT(*) FROM gold.authentication
WHERE status <> 'Success'
GROUP BY user.name
```

`tstats` is fast in Splunk because of the acceleration cache; in Lakewatch, performance comes
from Delta data-skipping and Z-order / liquid clustering on `time` and key fields. Don't promise
the user "same performance" — translate to the equivalent semantics, and let them tune the
underlying Delta table separately.

## Time, dates, and intervals

| SPL | Databricks SQL |
| --- | --- |
| `now()` | `current_timestamp()` |
| `earliest=-7d` | `INRANGE(time)` for interactive translations (default) — see *`INRANGE(time)`* below. Literal: `time >= current_timestamp() - INTERVAL 7 DAY` |
| `latest=now` | (the upper bound is implicit on the Lakewatch side; `INRANGE(time)` covers both ends) |
| `earliest=-1h@h` (snap to top of hour) | `INRANGE(time)` for interactive; literal: `time >= date_trunc('HOUR', current_timestamp() - INTERVAL 1 HOUR)` |
| `relative_time(now(), "-7d")` | `current_timestamp() - INTERVAL 7 DAY` (when you need a concrete timestamp; for the `WHERE` clause itself, prefer `INRANGE(time)`) |
| `strftime(_time, "%Y-%m-%d")` | `date_format(time, 'yyyy-MM-dd')` |
| `strptime("2026-01-01", "%Y-%m-%d")` | `to_timestamp('2026-01-01', 'yyyy-MM-dd')` |
| `bin _time span=1h` / `bucket _time span=1h` | `date_trunc('HOUR', time)` |
| `bin _time span=5m` | `from_unixtime(unix_timestamp(time) - unix_timestamp(time) % 300)` (Spark has no native 5-minute bin) |
| `_time` | `time` (canonical timestamp column in every Lakewatch silver/gold table) |
| *no `earliest`/`latest` in the SPL* | `INRANGE(time)` — see below |

### `INRANGE(time)` — bind to the UI timespan selector

Lakewatch's interactive SQL surfaces (the explore view, dashboard widgets) expose a **timespan
selector** at the top of the page. The SQL placeholder `INRANGE(time)` resolves at runtime to
the selected range.

**Default for SPL: use `INRANGE(time)`.**

Splunk's `earliest=`/`latest=` are different from KQL's `ago()` — they're almost always
**placeholder defaults** that the search-bar / dashboard time picker overrides in practice. A
Splunk search written as `earliest=-24h` is, in real-world use, run against whatever range the
user has selected in the time picker; the inline clause is rarely a deliberate pin. Translating
`earliest=-24h` literally to `time >= current_timestamp() - INTERVAL 24 HOUR` would lock the
Lakewatch query to a 24h window and remove the picker-driven flexibility — the opposite of what
the user wanted.

So the SPL skill defaults to `INRANGE(time)` for **all** time-range translations to interactive
contexts, and notes the original SPL window in mapping notes:

> The source SPL had `earliest=-24h`. I used `INRANGE(time)` so the Lakewatch explore-view
> timespan picker drives the lookback. If you actually wanted a hard 24h pin (e.g., for a
> scheduled rule), swap in `time >= current_timestamp() - INTERVAL 24 HOUR`.

**Use a literal `INTERVAL` only when:**

- The user explicitly says they're wrapping the result into a scheduled rule (rules don't get a
  UI picker — the lookback has to be concrete and tied to `schedule.atLeastEvery`).
- The user says they want a hard pin and not a UI-driven range.
- The SPL is from a SavedSearch / correlation rule that already has a hardcoded schedule (in
  which case the `earliest=` clause was deliberate).

**Why this differs from the KQL skill**: KQL's `ago()` is more often used as a deliberate pin in
Sentinel detection rules. SPL's `earliest=` is more often a placeholder. The two languages have
different cultural conventions around time-range expression; the translation defaults reflect
that.

## Strings, parsing, regex

| SPL | Databricks SQL |
| --- | --- |
| `len(s)` | `LENGTH(s)` |
| `lower(s)` | `LOWER(s)` |
| `upper(s)` | `UPPER(s)` |
| `substr(s, 1, 3)` | `SUBSTRING(s, 1, 3)` (Splunk is 1-indexed too) |
| `replace(s, "a", "b")` (literal) | `REPLACE(s, 'a', 'b')` |
| `replace(s, "(\d+)", "N")` (regex) | `REGEXP_REPLACE(s, '(\\d+)', 'N')` |
| `match(s, "regex")` | `regexp_like(s, 'regex')` |
| `rex field=s "(?<g>regex)"` | `regexp_extract(s, 'regex', 1) AS g` |
| `rex field=s mode=sed "s/foo/bar/g"` | `regexp_replace(s, 'foo', 'bar')` |
| `extract pairdelim=";" kvdelim="=" field=raw` | usually needs `regexp_extract_all` + struct projection. Surface as TODO if the SPL leans on it. |
| `split(s, ",")` | `SPLIT(s, ',')` |
| `mvjoin(arr, ",")` | `ARRAY_JOIN(arr, ',')` |
| `urldecode(s)` | `reflect('java.net.URLDecoder', 'decode', s, 'UTF-8')` (or `url_decode(s)` if DBR has it) |

**Regex gotcha**: SPL uses PCRE; Spark uses Java regex. Most patterns translate cleanly, but
named capture group syntax differs slightly (`(?<name>…)` in SPL → `(?<name>…)` in Spark too, but
referenced by index in `regexp_extract`, not by name).

### Command-line and switch matching

SPL queries that search Windows / EDR command lines for tool names and CLI switches are the most
common case where naive translations fail.

**Tool / executable names** (e.g., `powershell`). A loose `LIKE '%powershell%'` works but also
matches `mypowershellscript.txt`. The safer pattern:

```sql
regexp_like(LOWER(x), '\\bpowershell')      -- matches powershell, powershell.exe, powershell_ise.exe, renamed copies
```

**CLI switches** (e.g., `-enc`). A naive `LIKE '%-enc%'` matches `-encrypted`, `-enchant`, etc.
Use an anchored regex with explicit delimiters:

```sql
regexp_like(LOWER(x), '(^|[\\s"''])[-/]e(nc(odedcommand)?|c)(\\b|$)')
-- matches: -enc, -Enc, -ENC, -encodedcommand, -EncodedCommand, -ec, /EncodedCommand
-- does NOT match: -encrypted, -enchant
```

`\b` alone is too clever for shell tokens that begin with `-` or `/`.

## Multi-value fields and `mvexpand`

Splunk multi-value fields are arrays in OCSF; translate field-aware:

| SPL | Databricks SQL |
| --- | --- |
| `mvexpand foo` | `LATERAL VIEW EXPLODE(foo) AS foo` (or `SELECT explode(foo) AS foo`) |
| `mvcount(foo)` | `SIZE(foo)` |
| `mvindex(foo, 0)` | `foo[0]` |
| `mvindex(foo, -1)` | `foo[SIZE(foo) - 1]` |
| `mvfind(foo, "regex")` | No direct equivalent; use `FILTER(foo, x -> regexp_like(x, '…'))` and pick the first index |
| `mvfilter(match(foo, "regex"))` | `FILTER(foo, x -> regexp_like(x, '…'))` |
| `mvjoin(foo, ",")` | `ARRAY_JOIN(foo, ',')` |
| `mvappend(a, b)` | `CONCAT(a, b)` (works on arrays in Spark) |
| `mvdedup(foo)` | `ARRAY_DISTINCT(foo)` |

## `spath`, `xpath`, JSON paths

`spath` extracts a JSON path from `_raw` (or a named field):

| SPL | Databricks SQL |
| --- | --- |
| `spath path=user.name` | If `data` is VARIANT: `variant_get(data, '$.user.name', 'string')`. If the silver table promotes the field, use the promoted column directly. |
| `spath path=events{} output=ev` | `EXPLODE(variant_get(data, '$.events', 'array<variant>')) AS ev` |
| `spath input=raw_field` | `variant_get(raw_field, '$.…', '<type>')` — assuming `raw_field` is VARIANT |
| `xpath …` | XML parsing is fiddly in Spark. Use `xpath_string(xml, 'xpath_expression')` or pre-parse XML upstream. |

## Conditionals and `eval`

| SPL | Databricks SQL |
| --- | --- |
| `eval new = if(x > 0, "pos", "neg")` | `IF(x > 0, 'pos', 'neg') AS new` (or `CASE WHEN x > 0 THEN 'pos' ELSE 'neg' END`) |
| `eval new = case(x==1, "a", x==2, "b", 1==1, "c")` | `CASE WHEN x = 1 THEN 'a' WHEN x = 2 THEN 'b' ELSE 'c' END AS new` |
| `eval new = coalesce(a, b, c)` | `COALESCE(a, b, c) AS new` |
| `eval new = round(x, 2)` | `ROUND(x, 2) AS new` |
| `eval new = tostring(x, "commas")` | `FORMAT_NUMBER(x, 0) AS new` (for thousand separators) |
| `eval new = tonumber(s)` | `CAST(s AS DOUBLE) AS new` |
| `eval new = printf("%.2f%%", rate*100)` | `FORMAT_STRING('%.2f%%', rate * 100) AS new` |
| `eval new = strftime(_time, "%Y-%m-%d")` | `date_format(time, 'yyyy-MM-dd') AS new` |
| `eval new = strptime(s, "%Y-%m-%d %H:%M:%S")` | `to_timestamp(s, 'yyyy-MM-dd HH:mm:ss') AS new` |

## `timechart`, `chart`

`timechart` is `stats` plus an implicit `bin _time` plus a transpose:

```spl
... | timechart span=1h count by user
```

Equivalent to:

```sql
SELECT date_trunc('HOUR', time) AS hour, user, COUNT(*) AS n
FROM t
GROUP BY 1, 2
ORDER BY hour
```

If the original SPL had `timechart` because it was feeding a Splunk dashboard chart, the SQL
above feeds a Lakewatch dashboard widget the same way — Lakewatch's chart widget pivots on the
GROUP BY columns automatically. **Don't pivot manually with `PIVOT`** unless the original query
actually used `chart … by foo` in a way that requires a wide output.

## Splunk-only constructs to flag

These don't have a Lakewatch equivalent. Surface them in mapping notes and ask the user how to
resolve:

- `inputlookup foo.csv` — Splunk lookup CSV. Equivalent in Lakewatch is a Delta table the user
  needs to point you at.
- `lookup my_kvstore … OUTPUT …` — Splunk KV-store lookup. Same as above.
- `loadjob` / `savedsearch` / `populatelookup` — Splunk-only.
- `summarize` / `collect` / `tags` — Splunk-only.
- `iplocation` — geo-IP enrichment. Lakewatch has presets that compute `location.country` /
  `location.city` at ingest time, so OCSF rows often already have geo. If the SPL was using
  `iplocation` to add geo to non-geo'd events, point the user at the equivalent enrichment
  dataset.
- `cluster` (Splunk's pattern-discovery command) — no SQL equivalent; flag as a separate
  workflow (Lakewatch detection rules can use `ai_classify` for similar ML-style classification).
- ES `notable` macros — these expand to ES-specific SPL. Ask the user for the macro definition
  (`| rest /servicesNS/-/-/admin/macros`) and translate it explicitly.
- Custom search commands (anything not in the standard SPL command list) — case by case.
