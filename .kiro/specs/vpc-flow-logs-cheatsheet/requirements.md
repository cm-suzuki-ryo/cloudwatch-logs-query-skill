# Requirements Document

## Introduction

VPC Flow Logs バージョン11で追加された新フィールド（ネクストホップ系5フィールド、タグ系2フィールド、interface-type）を活用した CloudWatch Logs Insights クエリのチートシートを作成する。出力ファイル `vpc-flow-logs-v11-cheatsheet.md` をリポジトリルートに配置し、ユースケース／シナリオ別に整理された日本語の Markdown ドキュメントとする。

## Glossary

- **Cheatsheet**: VPC Flow Logs v11 フィールドを活用した CloudWatch Logs Insights クエリパターンをユースケース別にまとめた Markdown リファレンスドキュメント
- **VPC_Flow_Logs_v11**: VPC Flow Logs のカスタムフォーマットで利用可能なバージョン11の全フィールドセット（v2基本14フィールド＋拡張フィールド）
- **Next_Hop_Fields**: v11で追加されたネクストホップ関連5フィールド（next-hop-interface-id, next-hop-subnet-id, next-hop-az-id, next-hop-vpc-id, next-hop-interface-type）
- **Tag_Fields**: v11で追加されたEC2タグ関連2フィールド（instance-tag, interface-tag）
- **Legacy_Fields**: VPC Flow Logs v2 で定義された基本14フィールド（version, account-id, interface-id, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes, start, end, action, log-status）
- **Query_Builder_Skill**: 本リポジトリに含まれる CloudWatch Logs Query Builder スキルファイル（クエリ構文パターンの参照元）

## Requirements

### Requirement 1: ドキュメント配置と基本構造

**User Story:** As a AWS運用エンジニア, I want VPC Flow Logs v11 のクエリチートシートをリポジトリルートに単一ファイルとして参照できること, so that 必要なクエリパターンを素早く見つけて利用できる

#### Acceptance Criteria

1. THE Cheatsheet SHALL be written as a single Markdown file named `vpc-flow-logs-v11-cheatsheet.md` placed at the repository root directory
2. THE Cheatsheet SHALL be written entirely in Japanese (日本語)
3. THE Cheatsheet SHALL organize query examples by use case and scenario categories
4. THE Cheatsheet SHALL include a document title, introduction section, and table of contents linking to each use case section

### Requirement 2: フィールドリファレンステーブル

**User Story:** As a AWS運用エンジニア, I want VPC Flow Logs の全フィールド一覧を確認できること, so that クエリ作成時にフィールド名とデータ型を正確に参照できる

#### Acceptance Criteria

1. THE Cheatsheet SHALL include a field reference table covering all Legacy_Fields (v2 基本14フィールド)
2. THE Cheatsheet SHALL include a field reference table covering all Next_Hop_Fields (5フィールド)
3. THE Cheatsheet SHALL include a field reference table covering all Tag_Fields (2フィールド)
4. THE Cheatsheet SHALL include the interface-type field in the field reference table
5. WHEN listing fields in the reference table, THE Cheatsheet SHALL provide each field's name, data type, and a brief description in Japanese
6. THE Cheatsheet SHALL visually distinguish v11 new fields from Legacy_Fields in the field reference table using a separate section or clear labeling

### Requirement 3: クロスAZ通信検出クエリ

**User Story:** As a AWS運用エンジニア, I want クロスAZ通信を検出するクエリパターンを参照できること, so that AZ間データ転送コストの発生源を特定できる

#### Acceptance Criteria

1. THE Cheatsheet SHALL include a query example that detects cross-AZ communication by comparing next-hop-az-id values between source and destination
2. THE Cheatsheet SHALL include a query example that aggregates cross-AZ traffic volume (bytes) by source-destination AZ pair
3. THE Cheatsheet SHALL include a query example that identifies the top interfaces generating cross-AZ traffic using next-hop-interface-id and next-hop-az-id
4. WHEN presenting cross-AZ queries, THE Cheatsheet SHALL include a brief explanation of the scenario and expected output format

### Requirement 4: NAT Gateway ルーティング分析クエリ

**User Story:** As a AWS運用エンジニア, I want NAT Gateway 経由のトラフィックを分析するクエリパターンを参照できること, so that NAT Gateway の利用状況とコストへの影響を把握できる

#### Acceptance Criteria

1. THE Cheatsheet SHALL include a query example that identifies traffic routed through NAT Gateway by filtering on next-hop-interface-type equal to "nat-gateway"
2. THE Cheatsheet SHALL include a query example that aggregates NAT Gateway traffic volume by next-hop-vpc-id and next-hop-subnet-id
3. THE Cheatsheet SHALL include a query example that identifies top source addresses generating traffic through NAT Gateway
4. WHEN presenting NAT Gateway queries, THE Cheatsheet SHALL include a brief explanation of the scenario and expected output format

### Requirement 5: タグベース集計クエリ

**User Story:** As a AWS運用エンジニア, I want EC2タグを使ったトラフィック集計クエリを参照できること, so that アプリケーション単位やチーム単位でネットワーク利用状況を把握できる

#### Acceptance Criteria

1. THE Cheatsheet SHALL include a query example that aggregates traffic volume by instance-tag values
2. THE Cheatsheet SHALL include a query example that aggregates traffic volume by interface-tag values
3. THE Cheatsheet SHALL include a query example that combines tag-based grouping with action (ACCEPT/REJECT) status for security analysis
4. WHEN presenting tag-based queries, THE Cheatsheet SHALL include a brief explanation of the scenario and expected output format

### Requirement 6: 基本クエリパターン（従来フィールド）

**User Story:** As a AWS運用エンジニア, I want 従来フィールドのみで完結する基本的なクエリパターンも参照できること, so that v11新フィールドを使わない一般的な分析も同じチートシートで対応できる

#### Acceptance Criteria

1. THE Cheatsheet SHALL include a query example for Top Talkers analysis using srcaddr, dstaddr, and bytes fields from Legacy_Fields
2. THE Cheatsheet SHALL include a query example for REJECT action analysis using action, srcaddr, dstaddr, and dstport fields from Legacy_Fields
3. THE Cheatsheet SHALL clearly label these patterns as basic patterns using legacy fields only

### Requirement 7: クエリ構文の正確性

**User Story:** As a AWS運用エンジニア, I want チートシートのクエリが正しい Logs Insights QL 構文に従っていること, so that コピー＆ペーストですぐに実行できる

#### Acceptance Criteria

1. THE Cheatsheet SHALL use CloudWatch Logs Insights QL syntax for all query examples
2. THE Cheatsheet SHALL include `limit` clause in every query example (value between 50 and 100)
3. THE Cheatsheet SHALL include `sort` clause in query examples where result ordering is meaningful
4. THE Cheatsheet SHALL use `ispresent()` function before filtering or aggregating on optional fields
5. IF a query example uses time-based filtering, THEN THE Cheatsheet SHALL use `toMillis(@timestamp)` with epoch milliseconds instead of ISO 8601 string comparison

### Requirement 8: interface-type フィールド活用クエリ

**User Story:** As a AWS運用エンジニア, I want interface-type フィールドを使ったトラフィック分類クエリを参照できること, so that ENIの種類別にトラフィック特性を把握できる

#### Acceptance Criteria

1. THE Cheatsheet SHALL include a query example that aggregates traffic by interface-type to show distribution across ENI types (e.g., "amazon-elb", "nat_gateway", "efa")
2. THE Cheatsheet SHALL include a query example that combines interface-type with next-hop-interface-type for end-to-end routing path analysis
