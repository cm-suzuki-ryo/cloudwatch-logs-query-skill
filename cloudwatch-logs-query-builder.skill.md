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

### Use OpenSearch PPL when:
- Familiar with Unix pipe syntax
- Need `eval` for computed fields
- Need `trendline` for moving averages
- Need `grok` pattern parsing
- Need `fieldsummary` for field statistics

### Use OpenSearch SQL when:
- Familiar with SQL
- Need window functions (RANK, LAG, LEAD)
- Need complex JOINs or subqueries
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

### Time-series Rate & Time-shift (Logs Insights QL only)
```
# Per-minute error rate, compared with the same window 1 day earlier
filter @message like /(?i)error/
| stats count_over_time(@message) as errors by bin(1m) offset 1d

# Latency distribution as a histogram
stats histogram(@duration, 20) as latencyBuckets
```

### Cross Log Group Join
```
# Logs Insights QL
fields @timestamp, @message
| join @requestId [
    SOURCE '/aws/lambda/other-function'
    | fields @requestId, status
] as other
| display @timestamp, @message, other.status

# PPL
LEFT JOIN left=l, right=r ON l.requestId = r.requestId `other-log-group`
| fields l.`@message`, r.status

# SQL
SELECT A.`@message`, B.status
FROM `LogGroupA` as A
INNER JOIN `LogGroupB` as B ON A.requestId = B.requestId;
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
- `limit any` stops scanning early once enough results are found; `limit any N` fetches the first N results (cost reduction)
- A single query supports up to 10 `stats` commands
- `parse` has 4 modes — glob, regex, `logfmt`, `csv`. Add `multi` after a regex to emit one row per match; use `json field=` for chained JSON extraction
- Hashing functions (`md5`, `sha256`) work in `fields` and `filter` — useful to bucket or anonymize high-cardinality values
- Time-series functions (`rate`, `count_over_time`, `sum_over_time`, `histogram`) pair with `stats ... by bin()`; add `offset` to compare against a previous window
- SQL/PPL supports Standard Log Class only
- SOURCE/source command is CLI/API only (not available in console)
