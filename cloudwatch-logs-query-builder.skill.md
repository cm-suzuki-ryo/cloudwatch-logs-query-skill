# CloudWatch Logs Query Builder Skill

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

## Query Patterns

### Error Investigation
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

### Aggregation by Time Bucket
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

### Time-series Analytics (Logs Insights QL only)
```
# Count events per minute, with bin boundary shifted by offset
filter @message like /(?i)error/
| stats count_over_time(*) as errors by bin(1m) offset 5m

# Latency distribution — histogram is a `by`-clause grouping fn; 2nd arg is bucket WIDTH
filter ispresent(duration)
| stats count(*) as cnt by histogram(duration, 50)
```

### Operational Patterns (Logs Insights QL)
```
# API latency percentiles over time
filter ispresent(duration)
| stats avg(duration) as avgMs, pct(duration, 95) as p95Ms,
    pct(duration, 99) as p99Ms, max(duration) as maxMs by bin(5m)

# Lambda cold starts per hour
filter @type = "REPORT"
| parse @message /Init Duration: (?<initDuration>[\d\.]+)/
| filter ispresent(initDuration)
| stats count(*) as coldStarts, avg(initDuration) as avgInitMs by bin(1h)

# HTTP status code distribution
filter ispresent(statusCode)
| stats count(*) as requests by statusCode
| sort statusCode asc

# Trace a single request across streams
filter @requestId = "abc-123-def"
| fields @timestamp, @message, @logStream
| sort @timestamp asc

# Unique error messages
filter @message like /(?i)error/
| dedup @message
| limit 30
```

### Cross Log Group Join
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

### Subquery (Filter with Intermediate Results)
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

### Reduce Scan with Field Indexes
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

## Tips
- `bin()` time unit caps: ms=1000, s/m=60, h=24. Use `bin(5m)` not `bin(300s)` (capped to 60s)
- Backtick fields with special characters: `` `@message` ``, `` `Operation.Export` ``
- `limit any` stops scanning early once enough results are found; `limit any N` fetches the first N results (measured: 373 vs 36,535 records scanned for `limit any 3` vs `limit 3`)
- `stats` can be chained — up to 10 commands on Standard log class, up to 2 on Infrequent Access; later `stats` only sees fields from the previous one
- `parse` modes — glob, regex, `logfmt`, `csv`, `XML` (XPath). For regex multi-match add `multi` and use **named capture groups** (`as alias multi` is a syntax error); use `json field=` for chained JSON extraction
- `histogram(field, width)` is a `by`-clause grouping function — 2nd arg is bucket **width** (official docs say "number of buckets", but hands-on testing confirms width)
- Hashing functions (`md5`, `sha256`) work in `fields` and `filter` — useful to bucket or anonymize high-cardinality values
- `count_over_time(*)`/`sum_over_time(field)` equal `count(*)`/`sum(field)`; `offset` shifts bin boundary alignment. `rate(field, period)` computes the per-period rate of change within each bin (returns 0 when the field is constant)
- `strcontains` 3rd (case-insensitive) arg currently has no effect; `startsWith`/`endsWith` return `1`/`0`, not boolean
- `earliest(field)`/`latest(field)` return epoch **milliseconds** — convert with `fromMillis()`
- SQL/PPL supports Standard Log Class only
- SOURCE/source command is CLI/API only (not available in console)
- `join` supports only equality (`=`) conditions, max 1 join per query, max 50,000 unique keys on right side; subqueries on right side are not supported; right side `SOURCE` cannot contain pipe operations (Phase 1 limitation)
- `filter field in (subquery)` — subquery runs independently (max 30s); must produce exactly one output column (via `fields` or `stats`); nested/correlated subqueries are not supported
- When using `join` or subquery with `SOURCE`, select ONLY the primary log group in console; selecting both causes `ServiceUnavailableException`

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

### Post-Aggregation Filtering (HAVING Equivalent)

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

## Optimization Best Practices
- **Always cap results** with `limit` (50–100 typical) to control cost and avoid overwhelming output/agent context
- **Narrow the time range** as much as possible — it is the biggest lever on scanned volume and cost
- **Use `filterIndex` / `aws:fieldIndex`** on indexed fields to skip non-matching log groups
- **Parse JSON / extract fields early**, before filtering, so later commands operate on structured fields
- **Discover log groups first** (and confirm field names) before building complex queries
- **Build incrementally** — start simple, verify, then add aggregations and joins

## Common Pitfalls
- **Missing `limit`** — queries can return massive result sets
- **Not checking field existence** — use `ispresent(field)` before filtering/aggregating on optional fields
- **Case sensitivity** — field names and regex are case-sensitive; use `/(?i).../` for case-insensitive regex
- **Time format** — APIs expect ISO 8601 with timezone (e.g., `2026-06-10T10:00:00+00:00`)
- **Regex escaping** — wrap patterns in `/.../` and escape special characters
- **Overly wide time ranges** — slow and expensive

## AI Generation Anti-patterns

Common mistakes AI models make when generating CloudWatch Logs queries, and how to fix them.

### 1. Passing `parse`-created fields to `relevantfields`

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

### 2. Using PPL/SQL for Infrequent Access log groups

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

### 3. Using `bin(300s)` instead of `bin(5m)`

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

### 4. Placing `sort` after `dedup`

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

### 5. Using `unnest` on a `split()` result

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

### 6. Using dot access on `jsonParse` in the same `fields` command

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
