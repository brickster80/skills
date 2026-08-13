# KQL operator → Databricks SQL translation

This is the operator-level cheat sheet. Read the rows that apply to the query in front of you;
don't memorize the whole file.

## Table of contents

1. [Filtering and projection](#filtering-and-projection)
2. [Aggregations and `summarize`](#aggregations-and-summarize)
3. [Joins, unions, and lookups](#joins-unions-and-lookups)
4. [Time, dates, and intervals](#time-dates-and-intervals)
5. [Strings, parsing, regex](#strings-parsing-regex)
6. [Arrays, dynamic, and JSON](#arrays-dynamic-and-json)
7. [Window functions and `top-nested`](#window-functions-and-top-nested)
8. [Conditionals and case expressions](#conditionals-and-case-expressions)
9. [`make-series` and time-series](#make-series-and-time-series)
10. [KQL-only constructs to flag](#kql-only-constructs-to-flag)

---

## Filtering and projection

| KQL | Databricks SQL | Notes |
| --- | --- | --- |
| `where x == "foo"` | `WHERE x = 'foo'` | KQL `==` is `=` in SQL. Single-quote strings. |
| `where x != "foo"` | `WHERE x <> 'foo'` (or `!=`) | Both work in Spark. |
| `where x has "foo"` | `regexp_like(LOWER(x), '\\bfoo\\b')` (token / word-boundary match) — but see [Command-line and switch matching](#command-line-and-switch-matching) for the cases where `\b` does the wrong thing. | KQL `has` is a token match (Unicode word boundary), case-insensitive. `contains` is substring → `ILIKE '%foo%'`. |
| `where x contains "foo"` | `WHERE x ILIKE '%foo%'` | Case-insensitive by default in KQL. |
| `where x startswith "foo"` | `WHERE x ILIKE 'foo%'` | |
| `where x in ("a","b")` | `WHERE x IN ('a','b')` | |
| `where x !in~ ("a","b")` | `WHERE LOWER(x) NOT IN ('a','b')` | `~` suffix is case-insensitive in KQL. |
| `where isnotempty(x)` | `WHERE x IS NOT NULL AND x <> ''` | Empty string is not null in Spark. |
| `where isnull(x)` | `WHERE x IS NULL` | |
| `project a, b, c` | `SELECT a, b, c` | |
| `project-rename new = old` | `SELECT old AS new` | |
| `project-away x` | List the remaining columns explicitly. | Spark has no "all but" projection. |
| `extend y = x * 2` | Inline in `SELECT`: `SELECT x * 2 AS y, …` | Avoid CTE-per-`extend`. |
| `distinct a, b` | `SELECT DISTINCT a, b` | |
| `take 100` / `limit 100` | `LIMIT 100` | |

## Aggregations and `summarize`

| KQL | Databricks SQL |
| --- | --- |
| `summarize count() by user` | `SELECT user, COUNT(*) FROM t GROUP BY user` |
| `summarize n = count() by user` | `SELECT user, COUNT(*) AS n FROM t GROUP BY user` |
| `summarize dcount(ip) by user` | `SELECT user, COUNT(DISTINCT ip) FROM t GROUP BY user` |
| `summarize sum(bytes) by ip` | `SELECT ip, SUM(bytes) FROM t GROUP BY ip` |
| `summarize min(time), max(time) by user` | `SELECT user, MIN(time), MAX(time) FROM t GROUP BY user` |
| `summarize avg(latency) by service` | `SELECT service, AVG(latency) FROM t GROUP BY service` |
| `summarize percentile(latency, 95) by service` | `SELECT service, PERCENTILE_APPROX(latency, 0.95) FROM t GROUP BY service` |
| `summarize make_set(ip) by user` | `SELECT user, COLLECT_SET(ip) FROM t GROUP BY user` |
| `summarize make_list(ip) by user` | `SELECT user, COLLECT_LIST(ip) FROM t GROUP BY user` |
| `summarize arg_max(time, *) by user` | `SELECT * FROM t QUALIFY ROW_NUMBER() OVER (PARTITION BY user ORDER BY time DESC) = 1` |
| `summarize arg_min(time, *) by user` | Same as above with `ORDER BY time ASC`. |
| `summarize count() by bin(time, 1h), user` | `SELECT date_trunc('HOUR', time) AS hour, user, COUNT(*) FROM t GROUP BY 1, 2` |

**`countif` and conditional aggregates**:

| KQL | Databricks SQL |
| --- | --- |
| `summarize countif(status != 'Success')` | `SUM(CASE WHEN status <> 'Success' THEN 1 ELSE 0 END)` (or `COUNT_IF(status <> 'Success')` in DBR ≥ 14.0) |
| `summarize sumif(bytes, bytes > 0)` | `SUM(CASE WHEN bytes > 0 THEN bytes END)` |

**`top` / `top-nested`**:

```kql
T | top 10 by count_ desc
```

```sql
SELECT * FROM t ORDER BY count_ DESC LIMIT 10
```

`top-nested` (grouped top-N) is a window function:

```sql
SELECT * FROM t
QUALIFY ROW_NUMBER() OVER (PARTITION BY group_col ORDER BY count_ DESC) <= 10
```

## Joins, unions, and lookups

| KQL | Databricks SQL |
| --- | --- |
| `T1 | join kind=inner T2 on key` | `SELECT … FROM T1 INNER JOIN T2 ON T1.key = T2.key` |
| `T1 | join kind=leftouter T2 on key` | `LEFT JOIN` |
| `T1 | join kind=leftanti T2 on key` | `LEFT ANTI JOIN` (Spark supports this directly) |
| `T1 | join kind=leftsemi T2 on key` | `LEFT SEMI JOIN` |
| `T1 | union T2` | `SELECT … FROM T1 UNION ALL SELECT … FROM T2` (KQL `union` is UNION ALL, not deduped). |
| `T1 | lookup T2 on key` | A left join in Spark. KQL `lookup` is broadcast-friendly; mark the small side with `/*+ BROADCAST(T2) */` if needed. |

KQL `join` defaults to `innerunique` (deduplicates left side first). If the original query relied
on this, add a `DISTINCT` on the left's join key columns or use a CTE — and **call this out in
mapping notes** because forgetting it changes row counts.

## Time, dates, and intervals

| KQL | Databricks SQL |
| --- | --- |
| `now()` | `current_timestamp()` |
| `ago(7d)` | `current_timestamp() - INTERVAL 7 DAY` |
| `ago(30m)` | `current_timestamp() - INTERVAL 30 MINUTES` |
| `datetime(2025-01-01)` | `TIMESTAMP '2025-01-01'` |
| `startofday(time)` | `date_trunc('DAY', time)` |
| `startofhour(time)` | `date_trunc('HOUR', time)` |
| `bin(time, 1h)` | `date_trunc('HOUR', time)` |
| `bin(time, 5m)` | `from_unixtime(unix_timestamp(time) - unix_timestamp(time) % 300)` (Spark has no built-in 5-minute bin) |
| `datetime_diff('minute', t1, t2)` | `TIMESTAMPDIFF(MINUTE, t2, t1)` |
| `format_datetime(time, 'yyyy-MM-dd')` | `date_format(time, 'yyyy-MM-dd')` |
| `totimespan('00:05:00')` | `INTERVAL 5 MINUTES` |
| `time between (a .. b)` | `time BETWEEN a AND b` |
| *no time filter in the KQL* (Sentinel UI's timespan selector supplied it) | `INRANGE(time)` — see below |

**KQL durations** (`1d`, `5m`, `2h`) become **SQL intervals** (`INTERVAL 1 DAY`, `INTERVAL 5
MINUTES`, `INTERVAL 2 HOURS`). Note: Spark accepts both `MINUTE` and `MINUTES`.

### `INRANGE(time)` — bind to the UI timespan selector

Lakewatch's interactive SQL surfaces (the explore view, dashboard widgets) expose a **timespan
selector** at the top of the page. The SQL placeholder `INRANGE(time)` resolves at runtime to the
selected range — e.g., if the user picks "last 24 hours" in the UI, `INRANGE(time)` becomes
`time >= <now-24h> AND time <= <now>`.

When to use it:

- **The original KQL has no time filter** — the query relied on Sentinel / Defender's UI
  timespan picker. Translate the absence with `WHERE INRANGE(time) AND …` so the Lakewatch UI
  picker drives the same behavior. Don't invent a default like `INTERVAL 7 DAY` — that's a
  semantic change the user didn't ask for.
- **The user is building an interactive dashboard or explore-view query**, not a scheduled
  detection rule. Rules don't get a UI selector; they get a `schedule` block. For rule
  translations, keep the hardcoded interval and surface the lookback as a mapping note.

When the KQL **does** have a hardcoded time filter (`ago(7d)`, `between(...)`), translate it
literally — don't replace it with `INRANGE(time)`. The user already pinned the window on
purpose; replacing it would let the UI override their intent.

If in doubt, call it out in the mapping notes ("kept `INTERVAL 7 DAY` because the original KQL
pinned the lookback; switch to `INRANGE(time)` if you want this driven by the UI selector
instead").

## Strings, parsing, regex

| KQL | Databricks SQL |
| --- | --- |
| `strcat(a, b, c)` | `CONCAT(a, b, c)` |
| `strlen(s)` | `LENGTH(s)` |
| `tolower(s)` | `LOWER(s)` |
| `toupper(s)` | `UPPER(s)` |
| `substring(s, 0, 3)` | `SUBSTRING(s, 1, 3)` (SQL is 1-indexed!) |
| `replace_string(s, 'a', 'b')` | `REPLACE(s, 'a', 'b')` |
| `replace_regex(s, @'\d+', 'N')` | `REGEXP_REPLACE(s, '\\d+', 'N')` |
| `extract(@'(\d+)', 1, s)` | `REGEXP_EXTRACT(s, '(\\d+)', 1)` |
| `parse s with "user=" user " ip=" ip` | `SELECT REGEXP_EXTRACT(s, 'user=(\\S+) ip=(\\S+)', 1) AS user, REGEXP_EXTRACT(s, 'user=(\\S+) ip=(\\S+)', 2) AS ip` |
| `split(s, ',')` | `SPLIT(s, ',')` |
| `s matches regex @'…'` | `regexp_like(s, '…')` |

**Regex gotcha**: KQL string literals use `@'...'` to disable escaping. Spark SQL uses normal
string literals where backslashes need to be doubled — `\d+` in KQL becomes `\\d+` in SQL.

### Command-line and switch matching

KQL queries that search Defender / Windows command lines for tool names and CLI switches are the
most common case where the naïve `\b…\b` translation fails. Two patterns matter:

**Tool / executable names** (e.g., `powershell`). KQL's `has 'powershell'` matches `powershell`,
`powershell.exe`, `powershell_ise.exe`, `c:\windows\system32\powershell.exe`, etc., because all
those rows contain `powershell` as a token. `regexp_like(LOWER(x), '\\bpowershell\\b')` matches
the first three (the boundary between `l` and `.` / `_` / space fires), but you may want to
match **any** suffix variant (e.g., catch a renamed `powershell_v2.exe`). Drop the trailing `\b`:

```sql
regexp_like(LOWER(x), '\\bpowershell')      -- matches powershell, powershell.exe, powershell_ise.exe, powershell_v2.exe, …
```

**CLI switches** (e.g., `-enc`). KQL's `has '-enc'` matches `-enc`, `-Enc`, `-ENC` — but also
`-encodedcommand`, `/EncodedCommand`, `-ec`, because the `has` token boundary sits **before** the
`-` (Sentinel treats `-`, `/`, whitespace, and quotes as token separators). A naïve
`regexp_like(LOWER(x), '\\b-enc\\b')` is broken in two ways:

1. **`\b` doesn't fire between whitespace and `-`** (both non-word), so it never matches the
   leading boundary.
2. **`-enc` should also match `-EncodedCommand`** (PowerShell accepts the short and long forms
   interchangeably) — a strict `\b…\b` would miss it.

The right pattern is an explicit leading delimiter (`(^|[\\s"'])` accepts start of string,
whitespace, or quote) and a relaxed trailing match:

```sql
regexp_like(LOWER(x), '(^|[\\s"''])[-/]e(nc(odedcommand)?|c)(\\b|$)')
-- matches: -enc, -Enc, -ENC, -encodedcommand, -EncodedCommand, -ec, /EncodedCommand, /enc, /ec
-- does NOT match: -enchant, -encrypted, /enclosed
```

If the user only cares about the short form `-enc`, narrow it to
`(^|[\\s"''])[-/]enc(\\s|$|["'])` — still anchored on a real delimiter, but stricter on the
right side. The trade-off is detection coverage; surface it explicitly in the mapping notes.

**General rule of thumb**: for command-line searches, ask yourself "what tokens does the user
expect to match?" and translate to a regex that anchors on **explicit delimiters** at the start
(and only optionally at the end). `\b` alone is too clever — it works for natural-language tokens
but not for shell tokens that begin with `-` or `/`.

## Arrays, dynamic, and JSON

| KQL | Databricks SQL |
| --- | --- |
| `mv-expand col` | `LATERAL VIEW EXPLODE(col) AS col` (or `SELECT explode(col) AS col`) |
| `mv-expand bag` (dictionary) | `LATERAL VIEW EXPLODE(map_entries(bag)) t AS k, v` |
| `array_length(arr)` | `SIZE(arr)` |
| `array_index_of(arr, 'x')` | `ARRAY_POSITION(arr, 'x') - 1` (KQL is 0-indexed; `array_position` is 1-indexed) |
| `set_has_element(s, 'x')` | `ARRAY_CONTAINS(s, 'x')` |
| `parse_json(s)` | `FROM_JSON(s, '<schema>')` if you know the schema, otherwise `PARSE_JSON(s)` returns a VARIANT |
| `bag['key']` (dynamic) | `bag.key` (on struct) or `bag['key']` (on map) |
| `todynamic(s)` | `PARSE_JSON(s)` (returns VARIANT — use `variant_get` to reach fields) |

**Defender silver tables**: the source row payload lives in a VARIANT column called `data`. Reach
fields with `try_variant_get(data, '$.properties.<FieldName>', '<type>')` where `<type>` is
`'string'`, `'int'`, `'long'`, `'boolean'`, `'timestamp'`, etc.

## Window functions and `top-nested`

KQL has `prev()`, `next()`, and `serialize` for ordered row access. These map to Spark window
functions:

| KQL | Databricks SQL |
| --- | --- |
| `prev(x)` after `serialize | sort by t asc` | `LAG(x) OVER (ORDER BY t)` |
| `next(x)` after `serialize | sort by t asc` | `LEAD(x) OVER (ORDER BY t)` |
| `row_number()` after `partition by u | sort by t` | `ROW_NUMBER() OVER (PARTITION BY u ORDER BY t)` |

**Impossible-travel pattern** (common KQL → SQL conversion):

```kql
SigninLogs
| where ResultType == 0
| sort by UserPrincipalName, TimeGenerated asc
| serialize
| extend prev_country = prev(Location, 1), prev_time = prev(TimeGenerated, 1)
| where Location != prev_country and datetime_diff('minute', TimeGenerated, prev_time) <= 60
```

```sql
WITH t AS (
  SELECT
    user.name AS user_name,
    src_endpoint.location.country AS country,
    time,
    LAG(src_endpoint.location.country) OVER (PARTITION BY user.name ORDER BY time) AS prev_country,
    LAG(time) OVER (PARTITION BY user.name ORDER BY time) AS prev_time
  FROM gold.authentication
  WHERE status = 'Success'
)
SELECT * FROM t
WHERE country <> prev_country
  AND TIMESTAMPDIFF(MINUTE, prev_time, time) <= 60
```

## Conditionals and case expressions

| KQL | Databricks SQL |
| --- | --- |
| `iff(x > 0, 'pos', 'neg')` | `IF(x > 0, 'pos', 'neg')` (also `CASE WHEN x > 0 THEN 'pos' ELSE 'neg' END`) |
| `case(x == 1, 'a', x == 2, 'b', 'c')` | `CASE WHEN x = 1 THEN 'a' WHEN x = 2 THEN 'b' ELSE 'c' END` |
| `coalesce(a, b, c)` | `COALESCE(a, b, c)` |

## `make-series` and time-series

`make-series` produces dense time-series with gap filling. There is no single SQL equivalent —
the typical translation is a CTE that generates buckets via `sequence()`, joins to the data, and
fills nulls:

```kql
T
| make-series count() default=0 on TimeGenerated step 1h by user
```

```sql
WITH buckets AS (
  SELECT EXPLODE(sequence(
    date_trunc('HOUR', current_timestamp() - INTERVAL 24 HOUR),
    date_trunc('HOUR', current_timestamp()),
    INTERVAL 1 HOUR
  )) AS bucket
),
users AS (SELECT DISTINCT user.name AS user FROM t)
SELECT
  b.bucket,
  u.user,
  COALESCE(COUNT(t.user.name), 0) AS count_
FROM buckets b
CROSS JOIN users u
LEFT JOIN t
  ON date_trunc('HOUR', t.time) = b.bucket
  AND t.user.name = u.user
GROUP BY b.bucket, u.user
```

For detection rules you usually do **not** want `make-series` — gap-filled output is for charts.
Flag this in mapping notes if the user is building a rule.

## KQL-only constructs to flag

These don't have a Lakewatch equivalent. Surface them in mapping notes and ask the user how to
resolve:

- `_GetWatchlist("foo")` — Sentinel watchlist. Equivalent in Lakewatch is a delta table the user
  needs to point you at.
- `externaldata(...)` — inline external data source. Equivalent is a Lakewatch dataset (content
  pack) or a Unity Catalog table.
- `materialize()` — caching hint; drop it, Spark caches differently.
- Sentinel functions (e.g., `SecurityEvent_normalized`, `iff_v2`) — these are project-local
  functions; surface as TODO.
- `evaluate plugin(...)` (e.g., `bag_unpack`, `pivot`, `autocluster`) — case by case; some need
  `LATERAL VIEW` + `map_keys`, others have no direct equivalent.
