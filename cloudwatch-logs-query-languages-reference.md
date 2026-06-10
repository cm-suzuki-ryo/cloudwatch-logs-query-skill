# CloudWatch Logs Query Languages Reference

Last verified: 2026-06

CloudWatch Logs supports 3 query languages: Logs Insights QL, OpenSearch PPL, and OpenSearch SQL.

---

## 1. CloudWatch Logs Insights QL

Native query language for CloudWatch Logs. Uses pipe-separated commands.

### Auto-discovered Fields (@-prefixed)
CloudWatch automatically discovers these fields:

| Field | Description |
|-------|-------------|
| `@timestamp` | Log event timestamp |
| `@message` | Raw log message |
| `@logStream` | Log stream name |
| `@log` | Log group identifier |
| `@ingestionTime` | When CloudWatch received the event |

### Commands

| Command | Description |
|---------|-------------|
| `fields` | Display specific fields, supports functions to modify/create fields |
| `filter` | Filter log events by conditions |
| `filterIndex` | Force scan only indexed log groups (reduces scanned volume) |
| `stats` | Aggregate statistics (count, sum, avg, min, max, pct, etc.) with `by` grouping |
| `sort` | Sort results ascending (`asc`) or descending (`desc`) |
| `limit` | Limit results to N rows. `limit any` stops scanning early |
| `parse` | Extract fields. Modes: glob (`*`), regex (`/pattern/`), `logfmt`, `csv`, `XML` (XPath). Regex supports `multi` (named capture groups required); `json field=` for chained JSON extraction |
| `relevantfields` | Surface the fields most correlated with a condition. **`where` clause is required**: `relevantfields <fields> where <cond>`. Fields created by `parse` are NOT supported — use `@message` / auto-discovered fields only |
| `expand` | Flatten a list/array field into multiple records (one per element). Works on arrays extracted as strings via `parse` glob/regex. NOTE: a `split()` result is not flattened (no error, but stays one row); auto-parsed JSON list types may not expand |
| `display` | Display specific fields in output |
| `dedup` | Remove duplicates based on specified fields. NOTE: `sort` cannot follow `dedup` (syntax error) — order as `sort` → `dedup` |
| `pattern` | Auto-cluster log data into patterns |
| `diff` | Compare current time period with previous equal-length period. NOTE: only usable **after** `pattern` (e.g., `pattern @message \| diff`) |
| `anomaly` | Identify unusual patterns using ML |
| `unmask` | Show masked content (data protection) |
| `unnest` | Flatten a list into multiple records. Use on native arrays (e.g., from `jsonParse`). NOTE: a `split()` result raises `MalformedQueryException` |
| `lookup` | Enrich events with lookup table data |
| `join` | Combine events from different log groups by matching field |
| `subqueries` | Nested queries as input to another query |
| `addtotals` | Add a row-total column (default name `Total`) summing all numeric fields in the query; `addtotals fieldname=RowSum` renames it. Works after `fields`. NOTE: after `stats ... by`, the total column may not appear (the `by` field is treated as non-numeric). `col=true` adds a column-total row (console UI only) |
| `SOURCE` | Specify log groups (CLI/API only, not console) by prefix, account, class, data source, or tags |

### Syntax

```
fields @timestamp, @message
| filter @message like /Exception/
| stats count(*) as exceptionCount by bin(1h)
| sort exceptionCount desc
| limit 25
```

### Arithmetic Operators
`+`, `-`, `*`, `/`, `^` (exponent), `%` (modulus)

### Boolean Operators
`and`, `or`, `not`

### Comparison Operators
`=`, `!=`, `<`, `>`, `<=`, `>=`

### Filter Patterns
- `like /regex/` — regex match
- `like "string"` — glob match
- `not like` — negate
- `in [val1, val2]` — membership test
- `ispresent(field)` — field exists check

### Numeric Functions
| Function | Description |
|----------|-------------|
| `abs(a)` | Absolute value |
| `ceil(a)` | Round up |
| `floor(a)` | Round down |
| `greatest(a, ...)` | Largest value |
| `least(a, ...)` | Smallest value |
| `log(a)` | Natural log |
| `round(a [, d])` | Round to d decimal places |
| `sqrt(a)` | Square root |
| `haversine(lat1, lon1, lat2, lon2)` | Great-circle distance in km |
| `toNumber(field)` | String to number |
| `toInt(field)` | String to int (32-bit) |
| `toLong(field)` | String to long (64-bit) |
| `toDouble(field)` | String to double |

Conversion behavior (verified): `toInt`/`toLong` truncate the fractional part; `toDouble`/`toNumber` preserve decimals. A value that cannot be converted returns `null` (no error).

### Datetime Functions
| Function | Description |
|----------|-------------|
| `bin(period)` | Round @timestamp to period (e.g., `bin(5m)`, `bin(1h)`) |
| `datefloor(ts, period)` | Truncate timestamp to period |
| `dateceil(ts, period)` | Round up timestamp to period |
| `fromMillis(num)` | Epoch millis to timestamp |
| `toMillis(ts)` | Timestamp to epoch millis |
| `now()` | Current time in epoch seconds |

Time units: `ms`, `s`, `m`, `h`, `d`, `w`, `mo`/`mon`, `q`/`qtr`, `y`/`yr`

### General Functions
| Function | Description |
|----------|-------------|
| `ispresent(field)` | Returns true if field exists |
| `coalesce(f1, f2, ...)` | First non-null value |
| `case(cond1, val1, cond2, val2, ..., [default])` | Conditional (up to 10 branches) |
| `if(condition, trueVal, falseVal)` | If-then-else |

### JSON Functions
| Function | Description |
|----------|-------------|
| `jsonParse(field)` | Parse JSON string to map/list |
| `jsonStringify(field)` | Map/list to JSON string |

Access: `json.field`, `json.list[0]`, backticks for special chars: `` json.`special.key` ``

NOTE: dot access on a `jsonParse` result (`j.field`) does NOT work in the **same `fields` command** as the `jsonParse` call — split into two `fields` commands:
```
fields jsonParse(@message) as j
| fields j.status as statusCode, j.user as userName
```

### IP Address Functions
| Function | Description |
|----------|-------------|
| `isValidIp(field)` | Valid IPv4 or IPv6 |
| `isValidIpV4(field)` | Valid IPv4 |
| `isValidIpV6(field)` | Valid IPv6 |
| `isIpInSubnet(field, subnet)` | IP in CIDR range |
| `isIpv4InSubnet(field, subnet)` | IPv4 in subnet |
| `isIpv6InSubnet(field, subnet)` | IPv6 in subnet |
| `ipv4ToNumber(field)` | IPv4 string to number |
| `isPrivateIP(field)` | RFC 1918 private |
| `isPublicIP(field)` | Publicly routable |
| `isReservedIP(field)` | Reserved range |

### String Functions
| Function | Description |
|----------|-------------|
| `isempty(field)` | Missing or empty string |
| `isblank(field)` | Missing, empty, or whitespace only |
| `concat(s1, s2, ...)` | Concatenate strings |
| `ltrim(str [, chars])` | Trim left |
| `rtrim(str [, chars])` | Trim right |
| `trim(str [, chars])` | Trim both sides |
| `strlen(str)` | String length |
| `toupper(str)` | Uppercase |
| `tolower(str)` | Lowercase |
| `substr(str, start [, len])` | Substring |
| `replace(str, search, replace)` | Replace all occurrences |
| `regex_replace(str, pattern, replacement)` | Regex replace (RE2) |
| `strcontains(str, search [, caseInsensitive])` | Contains check. NOTE: the 3rd (case-insensitive) argument is accepted but had no effect in testing — matching stays case-sensitive |
| `startsWith(str, prefix)` | Starts with — returns `1`/`0` (not boolean) |
| `endsWith(str, suffix)` | Ends with — returns `1`/`0` (not boolean) |
| `urlencode(str)` / `urldecode(str)` | URL encode/decode |
| `base64encode(str)` / `base64decode(str)` | Base64 encode/decode |
| `split(str, delimiter)` | Split to array |

### Stats Aggregation Functions
- `count(*)`, `count(field)`, `count_distinct(field)`
- `sum(field)`, `avg(field)`, `min(field)`, `max(field)`
- `pct(field, percentile)` — e.g., `pct(@duration, 95)`
- `stddev(field)` — standard deviation
- `values(field)` — distinct values per group (returned as a comma-separated representation in API results)
- `earliest(field)`, `latest(field)` — return epoch **milliseconds** (not a human-readable timestamp); use `fromMillis()` to convert

`stats` can be chained: up to 10 `stats` commands on Standard log class, up to 2 on Infrequent Access. A later `stats` can only reference fields defined by the previous one; place `sort`/`limit` after the final `stats`.

### Hashing Functions
Usable in `fields` and `filter` commands.

| Function | Description |
|----------|-------------|
| `md5(field)` | MD5 hash of the string value |
| `sha256(field)` | SHA-256 hash of the string value |

### Time-series / Analytics Functions
Used with the `stats` command to analyze numeric fields over time windows.

| Function | Description |
|----------|-------------|
| `count_over_time(*)` | Count log events per time bin (use with `by bin(interval)`). Equivalent to `count(*)` |
| `sum_over_time(field)` | Sum field values per time bin (use with `by bin(interval)`). Equivalent to `sum(field)` |
| `rate(field, period)` | Per-interval rate of change for a numeric field. Use with `stats ... by bin()`; returns the average change per `period` within each bin (returns 0 when values are constant) |

**`histogram(field, bucketWidth)`** — a `by`-clause **grouping** function (not a `stats` aggregation). It buckets a numeric field by the given **bucket width** and returns each bucket's lower bound. Use as:
```
stats count(*) as cnt by histogram(value, 50)
```

**`offset` modifier** — append to a `stats ... by bin()` clause to shift the bin **boundary alignment** by a duration (bins are UTC-00:00 aligned by default). Useful to align buckets to business hours:
```
stats count(*) by bin(5m) offset 5m
```

**`rate` example** — average per-period change of a numeric field within each bin:
```
fields toMillis(@timestamp) as ms
| stats rate(ms, 1m) as rateVal by bin(5m)
```

### Sample Queries

**Last 25 events:**
```
fields @timestamp, @message | sort @timestamp desc | limit 25
```

**Exceptions per hour:**
```
filter @message like /Exception/
| stats count(*) as exceptionCount by bin(1h)
| sort exceptionCount desc
```

**Lambda memory usage:**
```
filter @type = "REPORT"
| stats max(@memorySize / 1000 / 1000) as provisionedMB,
    avg(@maxMemoryUsed / 1000 / 1000) as avgUsedMB,
    max(@maxMemoryUsed / 1000 / 1000) as maxUsedMB
```

**VPC flow logs — top transfers:**
```
stats sum(packets) as packetsTransferred by srcAddr, dstAddr
| sort packetsTransferred desc
| limit 15
```

**CloudTrail — events by service:**
```
stats count(*) by eventSource, eventName, awsRegion
```

**Parse example (glob):**
```
parse @message "user=*, method:*, latency := *" as @user, @method, @latency
| stats avg(@latency) by @method, @user
```

**Parse example (regex):**
```
parse @message /user=(?<user>.*?), method:(?<method>.*?), latency := (?<latency>.*?)/
| stats avg(latency) by @method, @user
```

**Parse example (logfmt):**
```
parse @message logfmt as lf
| filter lf.level = "error"
| display lf.msg, lf.duration
```

**Parse example (CSV):**
```
parse @message csv as timestamp, level, message
| filter level = "ERROR"
| display timestamp, message
```

**Parse example (regex multi-match — emit one row per match):**
```
# multi REQUIRES named capture groups; `as alias multi` is a syntax error
parse @message /(?<ip_addr>\d+\.\d+\.\d+\.\d+)/ multi
| stats count(*) by ip_addr
```

**Parse example (XML via XPath — one parse per field):**
```
filter @message like /<event>/
| parse @message XML '/event/level' as xlevel
| parse @message XML '/event/service' as xsvc
| display xlevel, xsvc
```

**Distinct values per group (`values`):**
```
filter ispresent(service) and ispresent(level)
| stats values(service) as services by level
```

**Parse example (chained JSON field extraction):**
```
parse @message /(?<payload>\{.*\})/ as payload
| json field=payload "user.name" as username
| display username
```

---

## 2. OpenSearch PPL (Piped Processing Language)

Unix pipe-based syntax. Commands are chained with `|`.

### Commands

| Command | Description | Example |
|---------|-------------|---------|
| `fields` | Project specific fields | `fields field1, field2` |
| `where` | Filter by condition | `where status="200"` |
| `stats` | Aggregations | `stats count(), avg(field) by group` |
| `parse` | Extract with regex | `parse field ".*/(?<name>[^/]+$)"` |
| `sort` | Sort (prefix `-` for desc) | `sort -fieldAlias` |
| `eval` | Create/modify fields | `eval newField = field * 2` |
| `rename` | Rename fields | `rename old as new` |
| `head` | Limit to first N rows | `head 20` |
| `top` | Most frequent values | `top 2 field by group` |
| `dedup` | Remove duplicates | `dedup field1` |
| `rare` | Least frequent values | `rare field1 by field2` |
| `join` | Join datasets | `LEFT JOIN left=l, right=r ON l.id = r.id \`lg\`` |
| `subquery` | Nested queries | `where f IN [search source=\`lg\` \| fields f2]` |
| `trendline` | Moving averages | `trendline sma(2, field) as alias` |
| `eventStats` | Enrich with summary stats | `eventstats sum(f1) by f2` |
| `expand` | Break multi-value to rows | `expand array_field as alias` |
| `fillnull` | Fill null values | `fillnull using field = "UNKNOWN"` |
| `flatten` | Flatten struct fields | `flatten struct as (a, b)` |
| `cidrmatch` | Check IP in CIDR | `where cidrmatch(ip, '10.0.0.0/8')` |
| `fieldsummary` | Basic stats per field | `fieldsummary includefields=f1 nulls=true` |
| `grok` | Parse with grok pattern | `grok email '.+@%{HOSTNAME:host}'` |
| `aws:fieldIndex` | Query indexed fields only | (see below) |

### Query Scope (SOURCE — CLI/API only)

**Log group:**
```
source = [lg:`/aws/lambda/my-function`] | where status = 200 | head 10
```

**Field index:**
```
source = [`aws:fieldIndex`="region", `region` = "us-west-2"] | where status = 200 | head 10
```

**Multiple field indexes:**
```
source = [`aws:fieldIndex` IN ("status", "region"), `status` = 200, `region` IN ("us-west-2", "us-east-1")] | head 10
```

**Data source:**
```
source = [ds:`data_source.type`] | where status = 200 | head 10
```

**Combined:**
```
search source=[
    ds:`DataSource1.Type1`,
    lg:`/aws/lambda/my-function-1`,
    `aws:fieldIndex` IN ("status"), `status` = 200
]
```

### Functions

**String:** LENGTH, LOWER, UPPER, TRIM, SUBSTRING, CONCAT, LIKE, REGEXP

**DateTime:** ADDDATE, DATE, DATE_FORMAT, DATEDIFF, DATE_SUB, TIMESTAMPADD, TIMESTAMPDIFF, CURRENT_TIMEZONE, UTC_TIMESTAMP, CURRENT_DATE

**Condition:** isnull, isnotnull, ifnull, nullif, IF, CASE

**Math:** ABS, ROUND, SQRT, POW, CEIL, FLOOR, ACOS, ASIN, ATAN, COS, SIN, TAN, LOG, LOG2, LOG10

**Crypto:** MD5, SHA1, SHA2

**JSON:** json_object, json_array, to_json_string, json_array_length, json_extract, json_keys, json_valid

### Restrictions
- Cannot use join or subquery with data source queries

---

## 3. OpenSearch SQL

Standard SQL syntax for querying CloudWatch Logs.

### Commands

| Command | Description | Example |
|---------|-------------|---------|
| `SELECT` | Project fields | `SELECT \`@message\`, Operation FROM \`LogGroupA\`` |
| `FROM` | Source table | (log group name in backticks) |
| `WHERE` | Filter conditions | `WHERE Operation = 'x'` |
| `GROUP BY` | Group for aggregation | `GROUP BY \`@logStream\`` |
| `HAVING` | Filter after grouping | `HAVING log_count > 100` |
| `ORDER BY` | Sort (ASC/DESC) | `ORDER BY \`@timestamp\` DESC` |
| `JOIN` | Join log groups | `INNER JOIN \`LogGroupB\` ON A.requestId = B.requestId` |
| `LIMIT` | Limit rows | `LIMIT 10` |
| `filterIndex` | Index-based filtering | `FROM \`filterIndex('region' = 'us-east-1')\`` |
| `logGroups` | Multiple log groups | `FROM \`logGroups(logGroupIdentifier: ['LG1','LG2'])\`` |
| `dataSource` | Multiple data sources | `FROM \`dataSource(['DS1', 'DS2'])\`` |

### Query Scope

**Log group:**
```sql
SELECT * FROM `logGroups(logGroupIdentifier: ['/aws/lambda/my-function'])`;
```

**Field index:**
```sql
SELECT * FROM `filterIndex('region' = 'us-east-1')`;
```

**Multiple indexes with IN:**
```sql
SELECT * FROM `filterIndex('status' = 200, 'region' IN ['us-east-1', 'us-west-2'])`;
```

**Data source:**
```sql
SELECT DS1.Column1 FROM `dataSource(['DataSource1.Type1', 'DataSource2.Type2'])` as DS1
WHERE DS1.Column1 = 'ABC';
```

**Combined:**
```sql
SELECT * FROM `
   logGroups(logGroupIdentifier: ['/aws/lambda/my-function'])
   filterIndex('status' = 200)
   dataSource(['DataSource1.Type1'])
`;
```

### Functions

**String:** upper, lower, trim, substring, concat, length, replace, regexp_extract, like

**Date:** current_date, date_add, date_format, datediff, last_day, current_timestamp

**Conditional:** IF, CASE WHEN, COALESCE, NULLIF, IFNULL

**Aggregate:** COUNT, SUM, AVG, MAX, MIN

**JSON:** get_json_object, from_json, to_json, json_tuple

**Array:** size, explode, array_contains

**Window:** ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, NTILE

**Conversion:** CAST, TO_DATE, TO_TIMESTAMP, BINARY

**Predicate:** IN, LIKE, BETWEEN, IS NULL, EXISTS, REGEXP

### Restrictions
- Only one JOIN per SELECT
- No JOIN or subqueries with data source queries
- Only one level of nested subqueries
- No multi-statement queries (no semicolon separation)
- Field names differing only by case not supported
- Functions must operate on field names (not literals alone)
- Use backticks for fields with special characters (`@message`, `Operation.Export`)

---

## Choosing a Language

| Criteria | Logs Insights QL | OpenSearch PPL | OpenSearch SQL |
|----------|-----------------|----------------|----------------|
| Best for | CloudWatch native users | Unix pipe familiarity | SQL familiarity |
| Pipe syntax | `\|` | `\|` | N/A (clauses) |
| JOIN support | Yes (cross log group) | Yes | Yes |
| Subqueries | Yes | Yes | Yes (1 level) |
| Pattern detection | Yes (`pattern`, `anomaly`) | No | No |
| Diff/compare | Yes (`diff`) | No | No |
| Window functions | No | No | Yes (RANK, LAG, LEAD) |
| Field indexes | `filterIndex` | `aws:fieldIndex` | `filterIndex` |
| Console support | Full | Full (except SOURCE) | Full (except SOURCE) |
| Log class | Standard + Infrequent Access* | Standard only | Standard only |

*Infrequent Access doesn't support `pattern`, `diff`, `unmask`.
