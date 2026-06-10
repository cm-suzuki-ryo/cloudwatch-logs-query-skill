# CloudWatch Logs Query Languages Skill

A Kiro skill and knowledge base for efficiently building CloudWatch Logs queries across all 3 supported query languages.

## Contents

| File | Description |
|------|-------------|
| `cloudwatch-logs-query-builder.skill.md` | Skill file — language selection guide, query patterns, tips |
| `cloudwatch-logs-query-languages-reference.md` | Knowledge base — full reference of commands, functions, and syntax |

## Supported Languages

1. **Logs Insights QL** — CloudWatch native, pipe-based. Best for console use, pattern detection, anomaly detection, and diff analysis.
2. **OpenSearch PPL** — Unix pipe-style. Best for eval/computed fields, trendlines, grok parsing.
3. **OpenSearch SQL** — Standard SQL. Best for JOINs, window functions (RANK, LAG, LEAD), subqueries.

## Quick Start

Ask the AI to build a query:
- "Write a Logs Insights query to find the top 10 error messages in the last hour"
- "Create a PPL query to calculate moving average of latency"
- "Write SQL to join API Gateway logs with Lambda logs by requestId"

The skill will select the appropriate language and produce optimized queries using the knowledge base as reference.

## Key Features

- **Language selection guidance** — picks the best language based on your needs
- **3-language parallel examples** — same logic expressed in all 3 languages
- **Field index optimization** — reduces scan volume and cost with `filterIndex` / `aws:fieldIndex`
- **Cross log group queries** — JOIN, subquery, and SOURCE patterns

## Quality Policy

This skill is maintained under the following principles:

1. **Only verified content** — All query patterns and function behaviors have been tested against a live AWS environment before inclusion.
2. **Always verify before updating** — Even official AWS documentation must be validated with real queries before being incorporated.
3. **Actively remove broken content** — Functions that don't work (e.g., `ago()`) or patterns that silently fail (e.g., ISO 8601 string comparison on `@timestamp`) are excluded and documented as pitfalls to prevent misuse.

## Source
Based on official AWS documentation (June 2026):
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData_PPL.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData_SQL.html

## Acknowledgements
Some operational best practices, common pitfalls, and practical query patterns
(e.g., latency percentiles, Lambda cold starts, request tracing, `stddev`) were
adapted from the **aws-observability** Kiro Power by AWS Labs:
- https://github.com/kirodotdev/powers/tree/main/aws-observability (Apache-2.0)

Verified runtime behavior for the May 2026 (13) and June 2026 (23) new
commands/functions — including corrections (e.g., `histogram` bucket width,
`parse multi` named-capture requirement, `relevantfields` requiring `where`,
`startsWith`/`endsWith` returning 1/0, `strcontains` 3rd arg, `rate` unconfirmed) —
is based on the author's own hands-on verification articles:
- https://dev.classmethod.jp/en/articles/cloudwatch-logs-insights-new-commands-functions-2026/ (May 2026 batch)
- https://dev.classmethod.jp/en/articles/cloudwatch-logs-insights-new-commands-functions-2026-june/ (June 2026 batch)

Query syntax and function coverage in this skill are maintained independently from
the official AWS documentation above.
