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

## Source
Based on official AWS documentation (June 2026):
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData_PPL.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData_SQL.html
