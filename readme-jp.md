# CloudWatch Logs クエリ言語スキル

CloudWatch Logsの3つのクエリ言語すべてに対応した、クエリ作成効率化のためのKiroスキル＆ナレッジベース。

## ファイル構成

| ファイル | 説明 |
|---------|------|
| `cloudwatch-logs-query-builder.skill.md` | スキルファイル — 言語選択ガイド、クエリパターン、Tips |
| `cloudwatch-logs-query-languages-reference.md` | ナレッジベース — コマンド・関数・構文の完全リファレンス |

## 対応言語

1. **Logs Insights QL** — CloudWatch固有のパイプ型言語。コンソール利用、パターン検出、異常検知、差分分析に最適。
2. **OpenSearch PPL** — Unixパイプスタイル。eval計算フィールド、トレンドライン、grokパースに最適。
3. **OpenSearch SQL** — 標準SQL。JOIN、ウィンドウ関数（RANK, LAG, LEAD）、サブクエリに最適。

## 使い方

AIにクエリ作成を依頼するだけ：
- 「直近1時間のエラーメッセージTop10を出すLogs Insightsクエリを書いて」
- 「レイテンシの移動平均を計算するPPLクエリを作って」
- 「API GatewayとLambdaのログをrequestIdでJOINするSQLを書いて」

スキルが最適な言語を選択し、ナレッジベースを参照して正確なクエリを生成します。

## 主な機能

- **言語選択ガイド** — 要件に応じて最適な言語を選択
- **3言語並列サンプル** — 同じロジックを3言語で表現
- **フィールドインデックス最適化** — `filterIndex` / `aws:fieldIndex` でスキャン量・コスト削減
- **クロスロググループクエリ** — JOIN、サブクエリ、SOURCEパターン

## ソース
AWS公式ドキュメント（2026年6月時点）に基づく：
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData_PPL.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData_SQL.html
