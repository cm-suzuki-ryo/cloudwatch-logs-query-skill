---
name: "cloudwatch-logs-query-builder"
displayName: "CloudWatch Logs Query Builder"
description: "Build optimal CloudWatch Logs queries across 3 languages (Logs Insights QL, OpenSearch PPL, OpenSearch SQL). Covers syntax, commands, functions, language selection, and query optimization."
keywords: ["cloudwatch", "logs", "query", "insights", "ppl", "sql", "log analysis", "filterIndex", "aws:fieldIndex", "observability", "log query", "cloudwatch logs insights"]
author: "because-and"
---

# CloudWatch Logs Query Builder Skill

> **Verification Policy:** This skill contains ONLY patterns verified against a live AWS environment. When updating, always run queries in a real environment before adding—even if the source is official AWS documentation. Remove any content confirmed to be broken or misleading.

## Description
Skill for efficiently building CloudWatch Logs queries. Covers syntax, commands, and functions for all 3 query languages (Logs Insights QL, OpenSearch PPL, OpenSearch SQL) and generates optimal queries based on user requirements.

## Quick Start
1. **Identify target:** Confirm log group name(s) and verify field names with `fields @message | limit 5`
2. **Select language:** Default to Logs Insights QL (use PPL for `eval`/`trendline`, SQL for window functions/JOIN)
3. **Build query:** `filter` → `parse` (if needed) → `stats ... by bin()` → `sort` → `limit`
- **MUST** set narrow time range via console/API first
- **MUST** always end with `limit 50–100`
- **Note:** If filter conditions depend on parsed fields, move `parse` before `filter`

## When to Use
- Writing CloudWatch Logs queries
- Optimizing log analysis queries
- Deciding which query language to use
- Reducing scan volume with filterIndex / aws:fieldIndex

## Critical Rules (MUST / MUST NOT)

> **Source of truth:** This table is the authoritative list of hard constraints. Other sections (Tips, Workflows, Anti-patterns, Error Scenarios) may repeat these rules inline for context — in case of conflict, this table wins.

| # | Rule | Rationale |
|---|------|-----------|
| 1 | **MUST** include `limit` (50–100) in every query | Unbounded results are expensive and can overwhelm agent context |
| 2 | **MUST** narrow the time range via console/API `startTime`/`endTime` first | Biggest cost lever — reduces scanned data volume |
| 3 | **MUST** use `toMillis(@timestamp)` with epoch milliseconds for in-query time filtering | ISO 8601 string comparison silently returns 0 results |
| 4 | **MUST NOT** use `ago()` | Does not exist — raises `MalformedQueryException` |
| 5 | **MUST** place `sort` before `dedup` | `dedup` followed by `sort` is a syntax error |
| 6 | **MUST** split `jsonParse` and dot access into separate `fields` commands | Dot access in the same command where `jsonParse` is called fails |
| 7 | **MUST** use `bin(5m)` not `bin(300s)` | `bin()` caps each unit at its natural max (s→60); large values are silently clamped |
| 8 | **MUST NOT** use PPL/SQL for Infrequent Access log class | IA supports Logs Insights QL only |
| 9 | **MUST NOT** pass `parse`-created fields to `relevantfields` | Only `@message` and auto-discovered `@`-prefixed fields are accepted |
| 10 | **MUST** select ONLY the primary log group in console when using `join`/subquery with SOURCE | Selecting both causes `ServiceUnavailableException` |
| 11 | **MUST** use `ispresent(field)` before filtering/aggregating on optional fields | Missing fields silently produce no results or skewed aggregations |

## Knowledge Base
- Context: CloudWatch Logs Query Languages Reference
- Path: ~/ws/projects/cloudwatch-logs-query-skill/cloudwatch-logs-query-languages-reference.md

## Priority Index
- **P1 基本** (毎回参照): 時刻指定 | filter | fields | sort/limit | parse
- **P2 標準** (高頻度): stats集計 | 時系列分析 | pattern/diff | display
- **P3 応用** (特定用途): JOIN | サブクエリ | anomaly | filterIndex
- **P4 特殊** (稀): dedup | expand/unnest | unmask | lookup | relevantfields | jsonParse | SOURCE | hashing

※ AIクエリ生成における参照頻度順に分類

## Language Selection Guide

### Use Logs Insights QL when:
- Using CloudWatch console interactively
- Need `pattern` for auto-clustering logs
- Need `diff` for time-period comparison
- Need `anomaly` for ML-based detection
- Need time-series analytics (`rate`, `count_over_time`, `sum_over_time`, `histogram`) or `offset` for time-shifted comparison
- Need to parse structured formats inline (`logfmt`, `csv`) or chained JSON extraction
- Need hashing (`md5`, `sha256`) to group/anonymize values
- Querying Infrequent Access log class
- Need cross-log-group `join` (inner/left) or `filter ... in (subquery)` with pipe syntax

### Use OpenSearch PPL when:
- Familiar with Unix pipe syntax
- Need `eval` for computed fields
- Need `trendline` for moving averages
- Need `grok` pattern parsing
- Need `fieldsummary` for field statistics

### Use OpenSearch SQL when:
- Familiar with SQL
- Need window functions (RANK, LAG, LEAD)
- Need standard SQL JOIN syntax (all 3 languages support JOIN; SQL is best for complex multi-table patterns)
- Need CAST / type conversion

---

## P1: 基本 (ほぼ毎回使う)

### 時刻指定 (Time Filtering)

Primary time range: always set via console UI time selector or API `startTime`/`endTime` (biggest cost lever). In-query narrowing uses `toMillis(@timestamp)` with epoch milliseconds for further filtering.

#### Examples

```
# Relative time — last 2 hours (now() returns epoch SECONDS, toMillis() returns MILLISECONDS)
fields @timestamp, @message
| filter toMillis(@timestamp) >= (now() * 1000 - 7200000)
| sort @timestamp desc
| limit 100

# Absolute time — epoch milliseconds
fields @timestamp, @message
| filter toMillis(@timestamp) > 1718020800000
| sort @timestamp desc
| limit 100
```

#### Tips
- `now()` returns epoch **seconds** — multiply by 1000 to compare with `toMillis()` (which returns ms)
- `ago()` does NOT exist — `filter @timestamp > ago(30m)` raises `MalformedQueryException`
- `@timestamp` cannot be compared to ISO 8601 strings (silently returns 0 results); use `toMillis(@timestamp)` with epoch milliseconds. APIs expect epoch seconds for `startTime`/`endTime`.
- `earliest(field)`/`latest(field)` return epoch **milliseconds** — convert with `fromMillis()`

---

### filter

#### Examples

```
# Logs Insights QL
fields @timestamp, @message
| filter @message like /(?i)(error|exception|fail)/
| sort @timestamp desc
| limit 100

# PPL
where `@message` like "%error%" or `@message` like "%exception%"
| fields `@timestamp`, `@message`
| sort - `@timestamp`
| head 100

# SQL
SELECT `@timestamp`, `@message` FROM `LogGroup`
WHERE `@message` LIKE '%error%' OR `@message` LIKE '%exception%'
ORDER BY `@timestamp` DESC LIMIT 100;
```

```
# Trace a single request across streams
filter @requestId = "abc-123-def"
| fields @timestamp, @message, @logStream
| sort @timestamp asc
```

#### Tips
- Case sensitivity: field names and regex are case-sensitive; use `/(?i).../` for case-insensitive regex
- `strcontains` 3rd (case-insensitive) arg currently has no effect; `startsWith`/`endsWith` return `1`/`0`, not boolean
- Not checking field existence: use `ispresent(field)` before filtering/aggregating on optional fields
- Regex escaping: wrap patterns in `/.../` and escape special characters
- `filter` after `stats` acts as HAVING — filters on aggregated aliases (e.g., `stats count(*) as c by svc | filter c > 5`)

#### ⚠️ AI Generation Pitfall: Using PPL/SQL for Infrequent Access log groups

**Mistake:**
```
-- PPL query targeting an IA log group
source = `my-ia-log-group`
| where status >= 500
```

**Why it is wrong:** Infrequent Access log class only supports Logs Insights QL. OpenSearch PPL and SQL are limited to Standard log class.

**Correct approach:**
```
fields @timestamp, @message
| filter status >= 500
```
Use Logs Insights QL syntax when the target log group is Infrequent Access.

---

### fields

#### Tips
- Backtick fields with special characters: `` `@message` ``, `` `Operation.Export` ``
- `@`-prefixed fields are auto-discovered by CloudWatch
- `relevantfields` はこの `@`-prefixed フィールドのみをサポート（P4 relevantfields セクション参照）

---

### sort / limit

#### Tips
- `limit any` stops scanning early once enough results are found; `limit any N` fetches the first N results (measured: 373 vs 36,535 records scanned for `limit any 3` vs `limit 3`)
- Always cap results with `limit` (50-100 typical) to control cost and avoid overwhelming output/agent context

---

### parse

#### Tips
- `parse` modes: glob, regex, `logfmt`, `csv`, `XML` (XPath). For regex multi-match add `multi` and use **named capture groups** (`as alias multi` is a syntax error); use `json field=` for chained JSON extraction

#### Examples

```
# Lambda cold starts per hour
filter @type = "REPORT"
| parse @message /Init Duration: (?<initDuration>[\d\.]+)/
| filter ispresent(initDuration)
| stats count(*) as coldStarts, avg(initDuration) as avgInitMs by bin(1h)
```

⚡ 常に limit を付与し、時間範囲を可能な限り狭めること（詳細は末尾 Optimization Best Practices 参照）

---

## P2: 標準 (頻繁に使う)

### stats 集計

#### Examples

```
# Logs Insights QL
stats count(*) as errorCount by bin(5m)

# PPL
stats count() as errorCount by span(`@timestamp`, 5m)

# SQL
SELECT COUNT(*) as errorCount,
  window_start FROM `LogGroup`
GROUP BY TUMBLE(`@timestamp`, '5 minutes');
```

```
# API latency percentiles over time
filter ispresent(duration)
| stats avg(duration) as avgMs, pct(duration, 95) as p95Ms,
    pct(duration, 99) as p99Ms, max(duration) as maxMs by bin(5m)

# HTTP status code distribution
filter ispresent(statusCode)
| stats count(*) as requests by statusCode
| sort statusCode asc
```

#### Post-Aggregation Filtering (HAVING Equivalent)

Logs Insights QL supports post-aggregation filtering by placing `filter` after `stats`. This works as a `HAVING` equivalent: the `filter` operates on the aliases produced by `stats`.

**Standard pattern: `stats ... by <group> | filter <aggAlias> <op> <val>`**

```
# Show only log streams with more than 3 events
stats count(*) as c by @logStream
| filter c > 3

# Show services with average latency above 200ms
filter ispresent(duration)
| stats avg(duration) as avgLatency by service
| filter avgLatency > 200

# Show time buckets where error rate exceeds threshold
filter @message like /(?i)error/
| stats count(*) as errors by bin(5m)
| filter errors > 50
| sort errors desc
```

**Key points:**
- The `filter` after `stats` can reference any alias defined in the preceding `stats` command
- You can combine it with `sort` and `limit` for further refinement (e.g., `| filter c > 3 | sort c desc | limit 10`)
- Multiple conditions work: `| filter errors > 10 and avgLatency > 100`

#### Tips
- `stats` can be chained — up to 10 commands on Standard log class, up to 2 on Infrequent Access; later `stats` only sees fields from the previous one
- SQL/PPL supports Standard Log Class only

#### ⚠️ AI Generation Pitfall: Using `bin(300s)` instead of `bin(5m)`

**Mistake:**
```
stats count(*) as cnt by bin(300s)
```

**Why it is wrong:** The `bin()` function caps each time unit at its natural maximum: seconds cap at 60. `bin(300s)` is silently clamped to `bin(60s)`, producing unexpected bucket sizes.

**Correct approach:**
```
stats count(*) as cnt by bin(5m)
```
Use the appropriate larger unit (minutes, hours) instead of large values of smaller units.

---

### 時系列分析 (Time-series Analytics)

Logs Insights QL only.

#### Examples

```
# Count events per minute, with bin boundary shifted by offset
filter @message like /(?i)error/
| stats count_over_time(*) as errors by bin(1m) offset 5m

# Latency distribution — histogram is a `by`-clause grouping fn; 2nd arg is bucket WIDTH
filter ispresent(duration)
| stats count(*) as cnt by histogram(duration, 50)
```

#### Tips
- `bin()` time unit caps: ms=1000, s/m=60, h=24. Use `bin(5m)` not `bin(300s)` (capped to 60s)
- `count_over_time(*)`/`sum_over_time(field)` equal `count(*)`/`sum(field)`; `offset` shifts bin boundary alignment. `rate(field, period)` computes the per-period rate of change within each bin (returns 0 when the field is constant)
- `histogram(field, width)` is a `by`-clause grouping function — 2nd arg is bucket **width** (official docs say "number of buckets", but hands-on testing confirms width)

---

### pattern / diff

`pattern` auto-clusters log messages. `diff` is only valid immediately after a `pattern` command for time-period comparison.

#### Tips
- `pattern` is pre-aggregation
- `diff` only after `pattern` (e.g., `pattern @message | diff`)

---

### display / addtotals

#### Tips
- `addtotals` after `fields`: adds a `Total` column. Works after `fields` but may not produce visible totals after `stats ... by`.
- `display` は post-aggregation コマンド。`stats` や `sort` の後に配置すること。

---

## P3: 応用 (特定ユースケース)

### JOIN

#### Examples

```
# Logs Insights QL — inner join
fields @timestamp, requestId, endpoint, status
| join type=inner left=app right=auth
    where app.requestId=auth.requestId
    (SOURCE '/aws/lambda/auth-service')
| fields app.requestId, app.endpoint, app.status, auth.authResult
| sort app.status desc
| limit 50

# Logs Insights QL — left join (retain unmatched left records)
fields @timestamp, requestId, endpoint, status
| join type=left left=order right=payment
    where order.transactionId=payment.transactionId
    (SOURCE '/aws/lambda/payment-service')
| filter !ispresent(payment.paymentStatus) or payment.paymentStatus != "SUCCESS"
| fields order.transactionId, order.orderId, payment.paymentStatus
| limit 50

# PPL
LEFT JOIN left=l, right=r ON l.requestId = r.requestId `other-log-group`
| fields l.`@message`, r.status

# SQL
SELECT A.`@message`, B.status
FROM `LogGroupA` as A
INNER JOIN `LogGroupB` as B ON A.requestId = B.requestId;
```

#### Tips
- `join` supports only equality (`=`) conditions, max 1 join per query, max 50,000 unique keys on right side; subqueries on right side are not supported; right side `SOURCE` cannot contain pipe operations (Phase 1 limitation)
- When using `join` or subquery with `SOURCE`, select ONLY the primary log group in console; selecting both causes `ServiceUnavailableException`

---

### サブクエリ (Subquery)

#### Examples

```
# Logs Insights QL — filter by values from another log group
filter requestId in (
    SOURCE '/aws/lambda/auth-service'
    | filter authResult = "FAILED"
    | fields requestId
)
| fields @timestamp, requestId, endpoint, status
| sort @timestamp desc
| limit 50

# Logs Insights QL — subquery with stats aggregation
filter requestId in (
    SOURCE '/aws/lambda/payment-service'
    | filter status = "FAILED"
    | stats count(*) as failureCount by requestId
    | filter failureCount > 3
    | fields requestId
)
| fields @timestamp, requestId, customerId, amount
| sort @timestamp asc
| limit 50
```

#### Tips
- `filter field in (subquery)` — subquery runs independently (max 30s); must produce exactly one output column (via `fields` or `stats`); nested/correlated subqueries are not supported

---

### anomaly

> いつ使うか: ログパターンの異常検知を自動で行いたい場合（時系列データに対して有効）

ML-based anomaly detection (Logs Insights QL only).

---

### filterIndex / fieldIndex

#### Examples

```
# Logs Insights QL
filterIndex region = 'us-east-1'
| filter status >= 500
| stats count(*) by service

# PPL
source = [`aws:fieldIndex`="region", `region` = "us-east-1"]
| where status >= 500
| stats count() by service

# SQL
SELECT service, COUNT(*) FROM `filterIndex('region' = 'us-east-1')`
WHERE status >= 500 GROUP BY service;
```

#### Tips
- Use `filterIndex` / `aws:fieldIndex` on indexed fields to skip non-matching log groups
- `filterIndex` is syntactically accepted at any position in the pipeline, but placing it first is strongly recommended for maximum scan reduction

---

## P4: 特殊 (稀に使う)

### dedup

#### Tips
- Correct order: `sort` then `dedup`. `dedup` cannot be followed by `sort` (syntax error).

#### Examples

```
# Unique error messages
filter @message like /(?i)error/
| dedup @message
| limit 30
```

#### ⚠️ AI Generation Pitfall: Placing `sort` after `dedup`

**Mistake:**
```
fields @timestamp, @message, service
| dedup service
| sort @timestamp desc
```

**Why it is wrong:** `dedup` cannot be followed by `sort` — this produces a syntax error. The correct order is `sort` first, then `dedup`.

**Correct approach:**
```
fields @timestamp, @message, service
| sort @timestamp desc
| dedup service
```

---

### expand / unnest

Neither command flattens a `split()` result. `unnest` on `split()` raises `MalformedQueryException`.

#### ⚠️ AI Generation Pitfall: Using `unnest` on a `split()` result

**Mistake:**
```
fields @message
| parse @message /tags=(?<tags>[^;]+)/
| fields split(tags, ",") as tagArray
| unnest tagArray
```

**Why it is wrong:** `unnest` on a `split()` result raises a `MalformedQueryException`. The `unnest` and `expand` commands do not support flattening arrays produced by `split()`.

**Correct approach:**
```
fields @message
| parse @message /tags=(?<tags>[^;]+)/
| parse tags /(?<tag1>[^,]+),?(?<tag2>[^,]+)?/
| filter ispresent(tag1)
| display tag1, tag2
```
Use multiple `parse` commands with regex capture groups to extract individual values from a delimited string, or filter against the original string using `like`/regex (e.g., `filter tags like /my-tag/`).

---

### unmask

> いつ使うか: データ保護でマスクされたフィールドを一時的に復元する必要がある場合

Data protection mask removal.

---

### lookup

> いつ使うか: 外部テーブル（CSV等）とログデータを結合する場合

Lookup table join.

---

### relevantfields

Requires a `where` clause. Only accepts `@message` and auto-discovered (`@`-prefixed) fields — `parse`-created fields are not supported.

#### ⚠️ AI Generation Pitfall: Passing `parse`-created fields to `relevantfields`

**Mistake:**
```
parse @message /user=(?<userId>\w+)/
| relevantfields userId where userId like /admin/
```

**Why it is wrong:** `relevantfields` only accepts `@message` and auto-discovered (`@`-prefixed) fields. Fields created by `parse` are not supported.

**Correct approach:**
```
relevantfields @message where @message like /admin/
```

---

### jsonParse

#### Tips
- Dot access on a `jsonParse` result cannot be in the same `fields` command; split into two `fields` commands.

#### ⚠️ AI Generation Pitfall: Using dot access on `jsonParse` in the same `fields` command

**Mistake:**
```
fields jsonParse(@message) as parsed, parsed.userId, parsed.action
```

**Why it is wrong:** Dot access on a `jsonParse` result cannot be used in the same `fields` command where `jsonParse` is called. The parsed object is not available until the next command in the pipeline.

**Correct approach:**
```
fields jsonParse(@message) as parsed
| fields parsed.userId, parsed.action, @timestamp
```
Split into two separate `fields` commands so the parsed object is available for dot access.

---

### SOURCE

CLI/API only (not available in console). Used to specify the target log group in `join` and subquery pipelines.

#### Tips
- `SOURCE` appears in the right side of `join` and inside `filter ... in (subquery)` to reference a different log group
- When selecting log groups in the console for a `join`/subquery query, select ONLY the primary log group; the `SOURCE` in the query handles the secondary group

#### Example

```
# Standalone usage (CLI/API only)
SOURCE '/aws/lambda/my-function'
| fields @timestamp, @message
| filter @message like /(?i)error/
| limit 50
```

See also: P3 JOIN セクション および P3 サブクエリ (Subquery) セクション

---

### hashing (md5, sha256)

#### Tips
- Hashing functions (`md5`, `sha256`) work in `fields` and `filter` — useful to bucket or anonymize high-cardinality values

---

## Common Workflows

### Workflow 1: Error Investigation
*(→ see also P1: filter for full 3-language examples)*
1. Set narrow time range via console UI or API `startTime`/`endTime`
2. (Optional) Use `filterIndex` if indexed fields are available to reduce scan
3. Filter errors: `filter @message like /(?i)(error|exception|fail)/`
4. Sort and cap: `sort @timestamp desc | limit 100`
5. If too many results, refine with `parse` to extract structured fields, then `stats count(*) by <field>`
- **MUST** include `limit` to control cost
- **MUST** narrow time range first (biggest cost lever)

### Workflow 2: Latency / Performance Analysis
1. Set time range (ideally ≤ 1h for initial analysis)
2. Confirm field existence: `filter ispresent(duration)`
3. Compute percentiles: `stats avg(duration) as avgMs, pct(duration, 95) as p95Ms, pct(duration, 99) as p99Ms by bin(5m)`
4. Identify outliers: `filter duration > <p99 threshold>` → examine `@message`
- **MUST** use `ispresent()` before aggregating optional fields
- **MUST** use `bin(5m)` not `bin(300s)` (capped to 60s)

### Workflow 3: Cross-Log-Group Correlation (Join)
*(→ see also P3: JOIN for full syntax examples)*
1. Select ONLY the primary log group in console
2. Build main pipeline: `fields @timestamp, requestId, endpoint, status`
3. Add join: `| join type=inner left=app right=auth where app.requestId=auth.requestId (SOURCE '/aws/lambda/auth-service')`
4. Display joined fields: `| fields app.requestId, app.endpoint, auth.authResult`
5. Cap results: `| limit 50`
- **MUST** select only primary log group (not both) — otherwise `ServiceUnavailableException`
- **MUST** use only equality (`=`) in join conditions
- Max 1 join per query, max 50,000 unique keys on right side

### Workflow 4: Cost-Optimized Aggregation
1. Apply `filterIndex` on indexed field first
2. Parse/filter to narrow rows early
3. Aggregate: `stats count(*) as cnt by <groupField>`
4. Post-filter (HAVING equivalent): `| filter cnt > <threshold>`
5. Sort and limit: `| sort cnt desc | limit 20`
- **MUST** place `filterIndex` first for maximum scan reduction
- **MUST** use post-aggregation `filter` (not `where`) for HAVING equivalent

### Workflow 5: Time-Series Comparison (diff)
*(→ see also P2: pattern / diff)*
1. Set time range covering both comparison periods
2. Cluster logs: `pattern @message`
3. Compare periods: `| diff`
4. Review pattern frequency changes across the two halves of the time range
- `diff` is ONLY valid immediately after `pattern`

---

## Command Pipeline Ordering Rules

Logs Insights QL uses a pipe-separated pipeline. Each command feeds its output to the next. The following rules define valid ordering.

### Valid Pipeline Flow

```
filterIndex (recommended first for scan reduction; accepted at any position)
    |
    v
fields / parse / filter / jsonParse  (pre-aggregation, any order, repeatable)
    |                            |
    |                            +---> relevantfields (requires where clause;
    |                            |       only @message/auto-discovered fields)
    |                            |
    |                            +---> pattern --> diff (diff only after pattern)
    |                            |
    |                            +---> addtotals (after fields; unreliable after stats...by)
    |
    v
stats ... by  (aggregation; chainable up to 10x Standard / 2x IA)
    |
    v
filter (post-aggregation: filters on aggregated aliases, acts as HAVING equivalent)
    |
    v
sort --> dedup (sort MUST precede dedup; dedup followed by sort = syntax error)
    |
    v
limit / display  (post-aggregation, after final stats/sort)

Parallel pipeline (join):
    main pipeline --+--> join @field [ SOURCE '...' | ... ] as alias
                    |
                    +--> display main.field, alias.field
```

> **Note:** This diagram covers the complete command set. Commands not shown here (`expand`, `unnest`) have restrictions — see Ordering Constraints below.

### Ordering Constraints

| Rule | Detail |
|------|--------|
| **`filterIndex` recommended first** | `filterIndex` is syntactically accepted at any position in the pipeline, but placing it first is strongly recommended for maximum scan reduction. When placed later, the scan-reduction benefit may be reduced or lost. |
| **`fields` / `parse` / `filter` before `stats`** | These pre-aggregation commands can appear in any order and repeat, but must come before the first `stats`. |
| **`stats` can chain** | Multiple `stats` commands can follow each other. Each subsequent `stats` only sees fields produced by the previous one. Standard log class allows up to 10 chained `stats`; Infrequent Access allows up to 2. |
| **`sort` before `dedup`** | `dedup` cannot be followed by `sort` (syntax error). Correct order: `sort` then `dedup`. |
| **`diff` only after `pattern`** | The `diff` command is only valid immediately after a `pattern` command (e.g., `pattern @message \| diff`). |
| **`pattern` is pre-aggregation** | `pattern` auto-clusters log messages. It sits in the pre-aggregation stage and can be followed by `diff` for time-period comparison. |
| **`relevantfields` restrictions** | Requires a `where` clause. Only accepts `@message` and auto-discovered (`@`-prefixed) fields — `parse`-created fields are not supported. |
| **`addtotals` after `fields`** | Adds a `Total` column. Works after `fields` but may not produce visible totals after `stats ... by`. |
| **`jsonParse` dot access requires separate command** | Dot access on a `jsonParse` result cannot be in the same `fields` command; split into two `fields` commands. |
| **`join` mid-pipeline** | `join` can appear after pre-aggregation commands. The sub-pipeline in brackets has its own `SOURCE`, `fields`, `parse`, `filter` chain. |
| **`expand`/`unnest` limitations** | Neither command flattens a `split()` result. `unnest` on `split()` raises `MalformedQueryException`. |
| **`sort` / `limit` / `display` after final `stats`** | Post-aggregation formatting commands come at the end of the pipeline, after the last `stats`. |
| **`filter` after `stats` acts as HAVING** | 詳細は P2 stats 集計セクションの Post-Aggregation Filtering を参照 |

### Example: Full Valid Pipeline

```
filterIndex accountId = '123456789012'
| fields @timestamp, @message
| parse @message /latency=(?<latency>\d+)/
| filter latency > 500
| stats avg(latency) as avgLatency, count(*) as cnt by bin(5m)
| stats max(avgLatency) as peakLatency by bin(1h)
| sort peakLatency desc
| dedup peakLatency
| limit 20
```

## Optimization Best Practices
- **Always cap results** with `limit` (50-100 typical) to control cost and avoid overwhelming output/agent context
- **Narrow the time range** as much as possible — it is the biggest lever on scanned volume and cost
- **Use `filterIndex` / `aws:fieldIndex`** on indexed fields to skip non-matching log groups
- **Parse JSON / extract fields early**, before filtering, so later commands operate on structured fields
- **Discover log groups first** (and confirm field names) before building complex queries
- **Build incrementally** — start simple, verify, then add aggregations and joins

## Common Pitfalls
- **Missing `limit`** — queries can return massive result sets
- **Not checking field existence** — use `ispresent(field)` before filtering/aggregating on optional fields
- **Case sensitivity** — field names and regex are case-sensitive; use `/(?i).../` for case-insensitive regex
- **Time format** — `@timestamp` cannot be compared to ISO 8601 strings (silently returns 0 results); use `toMillis(@timestamp)` with epoch milliseconds. APIs expect epoch seconds for `startTime`/`endTime`.
- **Regex escaping** — wrap patterns in `/.../` and escape special characters
- **Overly wide time ranges** — slow and expensive

## Error Scenarios

> For detailed code examples and corrections, see also the AI Generation Anti-patterns in the Knowledge Base reference.

| Symptom | Cause | Fix |
|---------|-------|-----|
| Query returns 0 results (no error) | ISO 8601 string comparison (`filter @timestamp > '2024-...'`) | Use `toMillis(@timestamp)` with epoch ms |
| Query returns 0 results (no error) | Wrong time range in console/API | Widen `startTime`/`endTime`, verify log group has data |
| Query returns 0 results (no error) | Field name typo or case mismatch | Run `fields @message | limit 5` first to confirm available fields |
| `MalformedQueryException` | Used `ago()` function | Replace with `toMillis(@timestamp) >= (now() * 1000 - <ms>)` |
| `MalformedQueryException` | `dedup` followed by `sort` | Reverse order: `sort` → `dedup` |
| `MalformedQueryException` | `unnest` on `split()` result | Use multiple `parse` with regex capture groups instead |
| `MalformedQueryException` | `parse ... as alias multi` | Use named capture groups: `parse ... /(?<name>...)/ multi` |
| `ServiceUnavailableException` | Selected multiple log groups with `join`/subquery SOURCE | Select ONLY the primary log group in console |
| Unexpected aggregation buckets | `bin(300s)` clamped to `bin(60s)` | Use `bin(5m)` — use appropriate larger unit |
| `relevantfields` returns no results | Passed `parse`-created field | Use only `@message` or auto-discovered `@`-prefixed fields |
| Post-`stats` filter has no effect | Referenced original field instead of alias | Use the alias from `stats`: `stats count(*) as c by svc | filter c > 5` |