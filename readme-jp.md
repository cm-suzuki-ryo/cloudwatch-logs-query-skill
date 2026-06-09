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

## 謝辞・出典
一部の運用ベストプラクティス、よくある落とし穴、実用クエリパターン
（レイテンシのパーセンタイル、Lambdaコールドスタート、リクエストトレース、`stddev` など）は、
AWS Labs による Kiro Power **aws-observability** を参考に取り込みました：
- https://github.com/kirodotdev/powers/tree/main/aws-observability （Apache-2.0）

2026年5月（13個）・6月（23個）の新コマンド・関数の実機挙動（`histogram` のバケット幅、
`parse multi` の名前付きキャプチャ必須、`relevantfields` の `where` 必須、
`startsWith`/`endsWith` の 1/0 返り値、`strcontains` 第3引数、`rate` 未確定などの補正を含む）は、
suzuki.ryo による実機検証記事に基づきます：
- https://dev.classmethod.jp/articles/cloudwatch-logs-insights-new-commands-functions-2026/ （5月分）
- https://dev.classmethod.jp/articles/cloudwatch-logs-insights-new-commands-functions-2026-june/ （6月分・公開予定）

なお、本スキルのクエリ構文・関数の網羅内容は、上記のAWS公式ドキュメントを一次情報として
独立に整備しています。
