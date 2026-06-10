# CloudWatch Logs Query Builder Skill

> **Verification Policy:** This skill contains ONLY patterns verified against a live AWS environment. When updating, always run queries in a real environment before adding--even if the source is official AWS documentation. Remove any content confirmed to be broken or misleading.

## Description
Skill for efficiently building CloudWatch Logs queries. Covers syntax, commands, and functions for all 3 query languages (Logs Insights QL, OpenSearch PPL, OpenSearch SQL) and generates optimal queries based on user requirements.

## When to Use
- Writing CloudWatch Logs queries
- Optimizing log analysis queries
- Deciding which query language to use
- Reducing scan volume with filterIndex / aws:fieldIndex

## Knowledge Base
- Context: CloudWatch Logs Query Languages Reference
- Path: ~/ws/projects/cloudwatch-logs-query-skill/cloudwatch-logs-query-languages-reference.md

## Priority Index
- **P1 基本** (毎回参照): 時刻指定 | filter | fields | sort/limit | parse
- **P2 標準** (高頻度): stats集計 | 時系列分析 | pattern/diff | display
- **P3 応用** (特定用途): JOIN | サブクエリ | anomaly | filterIndex
- **P4 特殊** (稀): dedup | expand/unnest | unmask | lookup | relevantfields | jsonParse | SOURCE | hashing

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
# Relative time -- last 2 hours (now() returns epoch SECONDS, toMillis() returns MILLISECONDS)
fields @timestamp, @message
| filter toMillis(@timestamp) >= (now() * 1000 - 7200000)
| sort @timestamp desc
| limit 100

# Absolute time -- epoch milliseconds
fields @timestamp, @message
| filter toMillis(@timestamp) > 1718020800000
| sort @timestamp desc
| limit 100
```

#### Tips
- `now()` returns epoch **seconds** -- multiply by 1000 to compare with `toMillis()` (which returns ms)
- `ago()` does NOT exist -- `filter @timestamp > ago(30m)` raises `MalformedQueryException`
- ISO 8601 string comparison does NOT work -- `filter @timestamp > '2024-01-01T00:00:00Z'` silently returns 0 results
- `earliest(field)`/`latest(field)` return epoch **milliseconds** -- convert with `fromMillis()`
- `@timestamp` cannot be compared to ISO 8601 strings (silently returns 0 results); use `toMillis(@timestamp)` with epoch milliseconds. APIs expect epoch seconds for `startTime`/`endTime`.

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

#### Tips
- Case sensitivity: field names and regex are case-sensitive; use `/(?i).../` for case-insensitive regex
- `strcontains` 3rd (case-insensitive) arg currently has no effect; `startsWith`/`endsWith` return `1`/`0`, not boolean
- Not checking field existence: use `ispresent(field)` before filtering/aggregating on optional fields
- Regex escaping: wrap patterns in `/.../` and escape special characters
- `filter` after `stats` acts as HAVING -- filters on aggregated aliases (e.g., `stats count(*) as c by svc | filter c > 5`)

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
- `stats` can be chained -- up to 10 commands on Standard log class, up to 2 on Infrequent Access; later `stats` only sees fields from the previous one
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

# Latency distribution -- histogram is a `by`-clause grouping fn; 2nd arg is bucket WIDTH
filter ispresent(duration)
| stats count(*) as cnt by histogram(duration, 50)
```

#### Tips
- `bin()` time unit caps: ms=1000, s/m=60, h=24. Use `bin(5m)` not `bin(300s)` (capped to 60s)
- `count_over_time(*)`/`sum_over_time(field)` equal `count(*)`/`sum(field)`; `offset` shifts bin boundary alignment. `rate(field, period)` computes the per-period rate of change within each bin (returns 0 when the field is constant)
- `histogram(field, width)` is a `by`-clause grouping function -- 2nd arg is bucket **width** (official docs say "number of buckets", but hands-on testing confirms width)

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

---

## P3: 応用 (特定ユースケース)

### JOIN

#### Examples

```
# Logs Insights QL -- inner join
fields @timestamp, requestId, endpoint, status
| join type=inner left=app right=auth
    where app.requestId=auth.requestId
    (SOURCE '/aws/lambda/auth-service')
| fields app.requestId, app.endpoint, app.status, auth.authResult
| sort app.status desc
| limit 50

# Logs Insights QL -- left join (retain unmatched left records)
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
# Logs Insights QL -- filter by values from another log group
filter requestId in (
    SOURCE '/aws/lambda/auth-service'
    | filter authResult = "FAILED"
    | fields requestId
)
| fields @timestamp, requestId, endpoint, status
| sort @timestamp desc
| limit 50

# Logs Insights QL -- subquery with stats aggregation
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
- `filter field in (subquery)` -- subquery runs independently (max 30s); must produce exactly one output column (via `fields` or `stats`); nested/correlated subqueries are not supported

---

### anomaly

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

# Trace a single request across streams
filter @requestId = "abc-123-def"
| fields @timestamp, @message, @logStream
| sort @timestamp asc
```

#### ⚠️ AI Generation Pitfall: Placing `sort` after `dedup`

**Mistake:**
```
fields @timestamp, @message, service
| dedup service
| sort @timestamp desc
```

**Why it is wrong:** `dedup` cannot be followed by `sort` -- this produces a syntax error. The correct order is `sort` first, then `dedup`.

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

Data protection mask removal.

---

### lookup

Lookup table join.

---

### relevantfields

Requires a `where` clause. Only accepts `@message` and auto-discovered (`@`-prefixed) fields -- `parse`-created fields are not supported.

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

CLI/API only (not available in console).

---

### hashing (md5, sha256)

#### Tips
- Hashing functions (`md5`, `sha256`) work in `fields` and `filter` -- useful to bucket or anonymize high-cardinality values

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

> **Note:** This diagram covers the complete command set. Commands not shown here (`expand`, `unnest`) have restrictions -- see Ordering Constraints below.

### Ordering Constraints

| Rule | Detail |
|------|--------|
| **`filterIndex` recommended first** | `filterIndex` is syntactically accepted at any position in the pipeline, but placing it first is strongly recommended for maximum scan reduction. When placed later, the scan-reduction benefit may be reduced or lost. |
| **`fields` / `parse` / `filter` before `stats`** | These pre-aggregation commands can appear in any order and repeat, but must come before the first `stats`. |
| **`stats` can chain** | Multiple `stats` commands can follow each other. Each subsequent `stats` only sees fields produced by the previous one. Standard log class allows up to 10 chained `stats`; Infrequent Access allows up to 2. |
| **`sort` before `dedup`** | `dedup` cannot be followed by `sort` (syntax error). Correct order: `sort` then `dedup`. |
| **`diff` only after `pattern`** | The `diff` command is only valid immediately after a `pattern` command (e.g., `pattern @message \| diff`). |
| **`pattern` is pre-aggregation** | `pattern` auto-clusters log messages. It sits in the pre-aggregation stage and can be followed by `diff` for time-period comparison. |
| **`relevantfields` restrictions** | Requires a `where` clause. Only accepts `@message` and auto-discovered (`@`-prefixed) fields -- `parse`-created fields are not supported. |
| **`addtotals` after `fields`** | Adds a `Total` column. Works after `fields` but may not produce visible totals after `stats ... by`. |
| **`jsonParse` dot access requires separate command** | Dot access on a `jsonParse` result cannot be in the same `fields` command; split into two `fields` commands. |
| **`join` mid-pipeline** | `join` can appear after pre-aggregation commands. The sub-pipeline in brackets has its own `SOURCE`, `fields`, `parse`, `filter` chain. |
| **`expand`/`unnest` limitations** | Neither command flattens a `split()` result. `unnest` on `split()` raises `MalformedQueryException`. |
| **`sort` / `limit` / `display` after final `stats`** | Post-aggregation formatting commands come at the end of the pipeline, after the last `stats`. |
| **`filter` after `stats` acts as HAVING** | `filter` placed after `stats` filters on aggregated aliases (e.g., `stats count(*) as c by svc \| filter c > 5`). This is the standard way to perform post-aggregation filtering in Logs Insights QL. |

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
- **Narrow the time range** as much as possible -- it is the biggest lever on scanned volume and cost
- **Use `filterIndex` / `aws:fieldIndex`** on indexed fields to skip non-matching log groups
- **Parse JSON / extract fields early**, before filtering, so later commands operate on structured fields
- **Discover log groups first** (and confirm field names) before building complex queries
- **Build incrementally** -- start simple, verify, then add aggregations and joins

## Common Pitfalls
- **Missing `limit`** -- queries can return massive result sets
- **Not checking field existence** -- use `ispresent(field)` before filtering/aggregating on optional fields
- **Case sensitivity** -- field names and regex are case-sensitive; use `/(?i).../` for case-insensitive regex
- **Time format** -- `@timestamp` cannot be compared to ISO 8601 strings (silently returns 0 results); use `toMillis(@timestamp)` with epoch milliseconds. APIs expect epoch seconds for `startTime`/`endTime`.
- **Regex escaping** -- wrap patterns in `/.../` and escape special characters
- **Overly wide time ranges** -- slow and expensive
