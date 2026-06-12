# Implementation Plan: VPC Flow Logs v11 チートシート

## Overview

リポジトリルートに `vpc-flow-logs-v11-cheatsheet.md` を作成する。VPC Flow Logs バージョン11 の新フィールド（ネクストホップ5フィールド、タグ2フィールド、interface-type）を活用した CloudWatch Logs Insights クエリパターンを日本語で整理したリファレンスドキュメントである。セクションごとに段階的に構築する。

## Tasks

- [x] 1. ドキュメントヘッダーとフィールドリファレンス作成
  - [x] 1.1 ドキュメントヘッダー・目次を作成
    - リポジトリルートに `vpc-flow-logs-v11-cheatsheet.md` を新規作成
    - H1 タイトル: `# VPC Flow Logs v11 CloudWatch Logs Insights クエリチートシート`
    - イントロダクション段落（目的、対象読者、前提条件を日本語で記述）
    - 目次（各ユースケースセクションへの GitHub 互換アンカーリンク）
    - _Requirements: 1.1, 1.2, 1.3, 1.4_

  - [x] 1.2 フィールドリファレンステーブルを作成
    - 従来フィールド（v2 基本14フィールド）テーブル: フィールド名・データ型・日本語説明の3列
    - v11 新規フィールドテーブル（ネクストホップ5フィールド）: next-hop-interface-id, next-hop-subnet-id, next-hop-az-id, next-hop-vpc-id, next-hop-interface-type
    - v11 新規フィールドテーブル（タグ2フィールド）: instance-tag, interface-tag
    - v11 新規フィールドテーブル（その他1フィールド）: interface-type
    - 従来フィールドと v11 新規フィールドを別セクションで明確にラベリング
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.6_

- [x] 2. クロスAZ通信検出クエリセクション作成
  - [x] 2.1 クロスAZ通信検出クエリ3パターンを作成
    - セクション概要（クロスAZ通信によるデータ転送コスト発生の背景説明を日本語で記述）
    - クエリ 3a: `next-hop-az-id` を使った基本的なクロスAZ通信検出（`ispresent()` + `az-id != next-hop-az-id` フィルタ、`limit 50`）
    - クエリ 3b: AZペア別トラフィック量集計（`stats sum(bytes)` by `az-id`, `next-hop-az-id`、`limit 50`）
    - クエリ 3c: クロスAZ通信量上位インターフェース（`stats sum(bytes)` by `next-hop-interface-id`, `az-id`, `next-hop-az-id`、`limit 50`）
    - 各クエリにシナリオ説明と期待される出力カラム構成を付記
    - すべてのクエリで `ispresent()` を v11 オプショナルフィールド使用前に適用
    - _Requirements: 3.1, 3.2, 3.3, 3.4, 7.1, 7.2, 7.3, 7.4_

- [x] 3. NAT Gateway ルーティング分析クエリセクション作成
  - [x] 3.1 NAT Gateway ルーティング分析クエリ3パターンを作成
    - セクション概要（NAT Gateway のコスト構造とデータ処理料金の説明を日本語で記述）
    - クエリ 4a: `next-hop-interface-type` が `"nat_gateway"` または `"regional_nat_gateway"` によるフィルタリング（アンダースコア区切り、ハイフンではない）
    - クエリ 4b: VPC/サブネット別 NAT Gateway トラフィック量集計（`stats sum(bytes)` by `next-hop-vpc-id`, `next-hop-subnet-id`、`limit 50`）
    - クエリ 4c: NAT Gateway 経由のトップ送信元アドレス（`stats sum(bytes)` by `srcaddr`、`limit 50`）
    - 各クエリにシナリオ説明と期待される出力カラム構成を付記
    - すべてのクエリで `ispresent()` を v11 オプショナルフィールド使用前に適用
    - **重要:** フィルタ値は `"nat_gateway"` と `"regional_nat_gateway"`（アンダースコア）を使用
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 7.1, 7.2, 7.3, 7.4_

- [x] 4. タグベース集計クエリセクション作成
  - [x] 4.1 タグベース集計クエリ3パターンを作成
    - セクション概要（タグベースのトラフィック可視化のメリットを日本語で記述）
    - クエリ 5a: `instance-tag` 別トラフィック集計（`ispresent()` + `stats sum(bytes)` by `instance-tag`、`limit 50`）
    - クエリ 5b: `interface-tag` 別トラフィック集計（`ispresent()` + `stats sum(bytes)` by `interface-tag`、`limit 50`）
    - クエリ 5c: タグ × action セキュリティ分析（`stats sum(bytes)` by `instance-tag`, `action`、`limit 50`）
    - 各クエリにシナリオ説明と期待される出力カラム構成を付記
    - すべてのクエリで `ispresent()` を v11 オプショナルフィールド使用前に適用
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 7.1, 7.2, 7.4_

- [x] 5. interface-type 別トラフィック分類と基本クエリパターン作成
  - [x] 5.1 interface-type 別トラフィック分類クエリ2パターンを作成
    - セクション概要（interface-type による ENI 種類とその用途の説明を日本語で記述）
    - クエリ 6a: `interface-type` 別トラフィック分布（`ispresent()` + `stats sum(bytes)` by `interface-type`、`limit 50`）
    - クエリ 6b: ソース → ネクストホップ ルーティングパス分析（`interface-type` × `next-hop-interface-type`、`limit 50`）
    - 各クエリにシナリオ説明と期待される出力カラム構成を付記
    - _Requirements: 8.1, 8.2, 7.1, 7.2, 7.4_

  - [x] 5.2 基本クエリパターン（従来フィールドのみ）2パターンを作成
    - セクションを「基本パターン（従来フィールドのみ）」と明記
    - クエリ 7a: Top Talkers 分析（`stats sum(bytes)` by `srcaddr`, `dstaddr`、`sort` + `limit 50`）
    - クエリ 7b: REJECT アクション分析（`filter action = "REJECT"` + `stats count(*)` by `srcaddr`, `dstaddr`, `dstport`、`sort` + `limit 50`）
    - _Requirements: 6.1, 6.2, 6.3, 7.1, 7.2, 7.3_

- [x] 6. 補足・注意事項セクション追加と最終確認
  - [x] 6.1 補足・注意事項セクションを作成
    - トラブルシューティングテーブル（問題・原因・対策の3列）
    - v11 フィールド未有効化時の挙動（クエリ0件）、`ispresent()` 未使用時の問題、時刻フィルタの正しい構文、クロスAZ判定の注意点、タグフィールド空の原因
    - _Requirements: 7.4, 7.5_

  - [x] 6.2 ドキュメント全体の整合性確認
    - 目次のアンカーリンクが全セクションに対応していることを確認
    - すべてのクエリに `limit` 句（50〜100）が含まれることを確認
    - すべてのv11オプショナルフィールド使用箇所に `ispresent()` があることを確認
    - NAT Gateway フィルタ値がアンダースコア（`nat_gateway`, `regional_nat_gateway`）であることを確認
    - Logs Insights QL 構文に準拠していることを確認
    - _Requirements: 7.1, 7.2, 7.3, 7.4, 7.5_

- [x] 7. Checkpoint - 最終確認
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- 本タスクの成果物はコードではなく単一の Markdown ドキュメント（`vpc-flow-logs-v11-cheatsheet.md`）である
- すべての説明文・見出しは日本語で記述する（フィールド名・コマンド名は英語のまま）
- クエリコードブロックは言語指定なし（` ``` `）で囲む（Logs Insights QL は SQL ではないため）
- NAT Gateway 関連のフィルタ値は必ずアンダースコア区切り: `"nat_gateway"`, `"regional_nat_gateway"`
- v11 新フィールドを中心にクエリ例を構成し、従来フィールドのみの基本パターンは補足的に含める
- 各クエリには `limit 50`〜`limit 100` を必須で含める
- v11 オプショナルフィールドの使用前には必ず `ispresent()` を適用する
- 参照ファイル: `cloudwatch-logs-query-builder.skill.md`（構築ルール）、`cloudwatch-logs-query-languages-reference.md`（構文リファレンス）

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1"] },
    { "id": 1, "tasks": ["1.2"] },
    { "id": 2, "tasks": ["2.1", "3.1", "4.1"] },
    { "id": 3, "tasks": ["5.1", "5.2"] },
    { "id": 4, "tasks": ["6.1"] },
    { "id": 5, "tasks": ["6.2"] }
  ]
}
```
