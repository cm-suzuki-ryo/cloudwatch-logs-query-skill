# Design Document: VPC Flow Logs v11 チートシート

## Introduction

本ドキュメントは `vpc-flow-logs-v11-cheatsheet.md` の技術設計を定義する。出力は単一の Markdown ファイルであり、VPC Flow Logs バージョン11 で追加されたフィールド（ネクストホップ系5フィールド、タグ系2フィールド、interface-type）を活用した CloudWatch Logs Insights クエリパターンをユースケース別に整理した日本語リファレンスドキュメントである。

---

## Architecture Overview

本成果物はコードではなく Markdown ドキュメントであるため、アーキテクチャはドキュメント構造そのものを指す。

```
vpc-flow-logs-v11-cheatsheet.md (リポジトリルート)
├── タイトル & イントロダクション
├── 目次（各セクションへのリンク）
├── フィールドリファレンス
│   ├── 従来フィールド（v2 基本14フィールド）
│   └── v11 新規フィールド（ネクストホップ5 + タグ2 + interface-type）
├── ユースケース別クエリセクション
│   ├── クロスAZ通信検出
│   ├── NAT Gateway ルーティング分析
│   ├── タグベース集計
│   ├── interface-type 別トラフィック分類
│   └── 基本クエリパターン（従来フィールドのみ）
└── 補足・注意事項
```

---

## Components

### Component 1: ドキュメントヘッダー

**責務:** ドキュメントのタイトル、概要説明、目次を提供する。

**構成要素:**
- H1 タイトル: `# VPC Flow Logs v11 CloudWatch Logs Insights クエリチートシート`
- イントロダクション段落: ドキュメントの目的、対象読者、前提条件を日本語で説明
- 目次: 各ユースケースセクションへの Markdown 内部リンク（`[セクション名](#アンカー)` 形式）

### Component 2: フィールドリファレンステーブル

**責務:** VPC Flow Logs の全フィールドを一覧表示し、クエリ作成時の参照とする。

**構成要素:**

#### 2a. 従来フィールドテーブル（v2 基本14フィールド）

| フィールド名 | データ型 | 説明 |
|---|---|---|
| version | INT | Flow Log バージョン |
| account-id | STRING | AWSアカウントID |
| interface-id | STRING | ネットワークインターフェースID |
| srcaddr | STRING | 送信元IPアドレス |
| dstaddr | STRING | 宛先IPアドレス |
| srcport | INT | 送信元ポート番号 |
| dstport | INT | 宛先ポート番号 |
| protocol | INT | IANAプロトコル番号 |
| packets | LONG | パケット数 |
| bytes | LONG | バイト数 |
| start | LONG | キャプチャ開始時刻（UNIXタイム） |
| end | LONG | キャプチャ終了時刻（UNIXタイム） |
| action | STRING | ACCEPT または REJECT |
| log-status | STRING | ログ記録ステータス |

#### 2b. v11 新規フィールドテーブル

**ネクストホップ系フィールド（5フィールド）:**

| フィールド名 | データ型 | 説明 |
|---|---|---|
| next-hop-interface-id | STRING | ネクストホップのネットワークインターフェースID |
| next-hop-subnet-id | STRING | ネクストホップのサブネットID |
| next-hop-az-id | STRING | ネクストホップのアベイラビリティゾーンID |
| next-hop-vpc-id | STRING | ネクストホップのVPC ID |
| next-hop-interface-type | STRING | ネクストホップのインターフェースタイプ（nat_gateway, regional_nat_gateway, vpc-endpoint 等） |

**タグ系フィールド（2フィールド）:**

| フィールド名 | データ型 | 説明 |
|---|---|---|
| instance-tag | STRING | EC2インスタンスに付与されたタグ値 |
| interface-tag | STRING | ENIに付与されたタグ値 |

**その他（1フィールド）:**

| フィールド名 | データ型 | 説明 |
|---|---|---|
| interface-type | STRING | ENIのタイプ（amazon-elb, nat_gateway, efa 等） |

**テーブル設計方針:**
- 従来フィールドと v11 新規フィールドは別セクション（別テーブル）で提示し、明確にラベリングする
- 各テーブルの列は「フィールド名」「データ型」「説明（日本語）」の3列構成
- v11 新規フィールドセクションにはさらにサブカテゴリ（ネクストホップ / タグ / その他）を設ける

### Component 3: クロスAZ通信検出クエリセクション

**責務:** AZ間データ転送コストの発生源を特定するためのクエリパターンを提供する。

**構成要素:**
- セクション概要（シナリオ説明）: クロスAZ通信が発生するとデータ転送コストが発生する背景を日本語で説明
- クエリ例 3a: `next-hop-az-id` を使った基本的なクロスAZ通信検出
- クエリ例 3b: AZペア別のトラフィック量集計（bytes）
- クエリ例 3c: クロスAZ通信量上位インターフェース特定
- 各クエリには期待される出力カラム構成を記載

**クエリ設計パターン:**

```
# 3a: クロスAZ通信検出（基本）
filter ispresent(`next-hop-az-id`)
| filter `az-id` != `next-hop-az-id`
| fields @timestamp, `interface-id`, `srcaddr`, `dstaddr`, `az-id`, `next-hop-az-id`, bytes
| sort bytes desc
| limit 50
```

```
# 3b: AZペア別トラフィック量集計
filter ispresent(`next-hop-az-id`)
| filter `az-id` != `next-hop-az-id`
| stats sum(bytes) as totalBytes, count(*) as flowCount by `az-id`, `next-hop-az-id`
| sort totalBytes desc
| limit 50
```

```
# 3c: クロスAZ通信量上位インターフェース
filter ispresent(`next-hop-az-id`) and ispresent(`next-hop-interface-id`)
| filter `az-id` != `next-hop-az-id`
| stats sum(bytes) as totalBytes by `next-hop-interface-id`, `az-id`, `next-hop-az-id`
| sort totalBytes desc
| limit 50
```

### Component 4: NAT Gateway ルーティング分析クエリセクション

**責務:** NAT Gateway 経由のトラフィック利用状況とコストへの影響を把握するためのクエリパターンを提供する。

**構成要素:**
- セクション概要（シナリオ説明）: NAT Gateway のコスト構造（データ処理料金）を日本語で説明
- クエリ例 4a: `next-hop-interface-type` が `"nat_gateway"` または `"regional_nat_gateway"` によるフィルタリング
- クエリ例 4b: VPC/サブネット別 NAT Gateway トラフィック量集計
- クエリ例 4c: NAT Gateway 経由のトップ送信元アドレス
- 各クエリには期待される出力カラム構成を記載

**クエリ設計パターン:**

注: `next-hop-interface-type` の NAT Gateway 関連の値はアンダースコア区切り:
- `"nat_gateway"` — AZ単位の NAT Gateway
- `"regional_nat_gateway"` — リージョナル NAT Gateway

```
# 4a: NAT Gateway 経由トラフィック検出（AZ単位 + リージョナル両方）
filter ispresent(`next-hop-interface-type`)
| filter `next-hop-interface-type` = "nat_gateway" or `next-hop-interface-type` = "regional_nat_gateway"
| fields @timestamp, `srcaddr`, `dstaddr`, `next-hop-interface-id`, `next-hop-interface-type`, bytes, packets
| sort bytes desc
| limit 50
```

```
# 4b: VPC/サブネット別 NAT Gateway トラフィック集計
filter ispresent(`next-hop-interface-type`)
| filter `next-hop-interface-type` = "nat_gateway" or `next-hop-interface-type` = "regional_nat_gateway"
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets by `next-hop-vpc-id`, `next-hop-subnet-id`, `next-hop-interface-type`
| sort totalBytes desc
| limit 50
```

```
# 4c: NAT Gateway 経由トップ送信元
filter ispresent(`next-hop-interface-type`)
| filter `next-hop-interface-type` = "nat_gateway" or `next-hop-interface-type` = "regional_nat_gateway"
| stats sum(bytes) as totalBytes, count(*) as flowCount by srcaddr
| sort totalBytes desc
| limit 50
```

### Component 5: タグベース集計クエリセクション

**責務:** EC2 タグを活用したアプリケーション/チーム単位でのネットワーク利用状況把握クエリを提供する。

**構成要素:**
- セクション概要（シナリオ説明）: タグベースのトラフィック可視化のメリットを日本語で説明
- クエリ例 5a: `instance-tag` によるトラフィック集計
- クエリ例 5b: `interface-tag` によるトラフィック集計
- クエリ例 5c: タグ + action（ACCEPT/REJECT）のセキュリティ分析
- 各クエリには期待される出力カラム構成を記載

**クエリ設計パターン:**

```
# 5a: instance-tag 別トラフィック集計
filter ispresent(`instance-tag`)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by `instance-tag`
| sort totalBytes desc
| limit 50
```

```
# 5b: interface-tag 別トラフィック集計
filter ispresent(`interface-tag`)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by `interface-tag`
| sort totalBytes desc
| limit 50
```

```
# 5c: タグ × action セキュリティ分析
filter ispresent(`instance-tag`)
| stats sum(bytes) as totalBytes, count(*) as flowCount by `instance-tag`, action
| sort totalBytes desc
| limit 50
```

### Component 6: interface-type 別トラフィック分類クエリセクション

**責務:** ENI種類別のトラフィック特性把握、およびソース→ネクストホップのルーティング経路分析クエリを提供する。

**構成要素:**
- セクション概要（シナリオ説明）: interface-type による ENI 種類とその用途を日本語で説明
- クエリ例 6a: `interface-type` 別トラフィック分布
- クエリ例 6b: `interface-type` × `next-hop-interface-type` エンドツーエンドルーティングパス分析

**クエリ設計パターン:**

```
# 6a: interface-type 別トラフィック分布
filter ispresent(`interface-type`)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by `interface-type`
| sort totalBytes desc
| limit 50
```

```
# 6b: ソース → ネクストホップ ルーティングパス分析
filter ispresent(`interface-type`) and ispresent(`next-hop-interface-type`)
| stats sum(bytes) as totalBytes, count(*) as flowCount by `interface-type`, `next-hop-interface-type`
| sort totalBytes desc
| limit 50
```

### Component 7: 基本クエリパターンセクション（従来フィールドのみ）

**責務:** v11 新フィールドを使用しない一般的な分析パターンを提供する。

**構成要素:**
- セクションラベル: 「基本パターン（従来フィールドのみ）」と明記
- クエリ例 7a: Top Talkers 分析（srcaddr, dstaddr, bytes）
- クエリ例 7b: REJECT アクション分析（action, srcaddr, dstaddr, dstport）

**クエリ設計パターン:**

```
# 7a: Top Talkers
stats sum(bytes) as totalBytes by srcaddr, dstaddr
| sort totalBytes desc
| limit 50
```

```
# 7b: REJECT 分析
filter action = "REJECT"
| stats count(*) as rejectCount by srcaddr, dstaddr, dstport
| sort rejectCount desc
| limit 50
```

---

## Interfaces

### 参照元インターフェース

本チートシートは以下のリポジトリ内ファイルを構文リファレンスとして参照する:

| 参照ファイル | 用途 |
|---|---|
| `cloudwatch-logs-query-builder.skill.md` | クエリ構築ルール、Critical Rules（limit 必須、ispresent() 使用等） |
| `cloudwatch-logs-query-languages-reference.md` | Logs Insights QL の全コマンド・関数・構文リファレンス |

### クエリ構文ルール（スキルファイルより適用）

チートシートのすべてのクエリ例に適用する必須ルール:

1. **`limit` 必須**: すべてのクエリに `limit 50` ～ `limit 100` を含める
2. **`ispresent()` 使用**: オプショナルフィールド（v11 新フィールドを含む）でフィルタ/集計する前に `ispresent()` でフィールド存在確認を行う
3. **時刻フィルタリング**: クエリ内で時間絞り込みを行う場合は `toMillis(@timestamp)` とエポックミリ秒を使用する（ISO 8601 文字列比較は不可）
4. **Logs Insights QL 構文**: すべてのクエリ例は CloudWatch Logs Insights QL で記述する
5. **`sort` 配置**: 結果の順序が意味を持つクエリには `sort` を含める（降順の場合 `desc`）

---

## Data Models

### VPC Flow Log レコードモデル

本チートシートが扱う VPC Flow Log レコードのフィールド構成:

```
VPCFlowLogRecord {
  // 従来フィールド (v2 基本14フィールド)
  version: INT
  account-id: STRING
  interface-id: STRING
  srcaddr: STRING
  dstaddr: STRING
  srcport: INT
  dstport: INT
  protocol: INT
  packets: LONG
  bytes: LONG
  start: LONG          // UNIXタイムスタンプ（秒）
  end: LONG            // UNIXタイムスタンプ（秒）
  action: STRING       // "ACCEPT" | "REJECT"
  log-status: STRING   // "OK" | "NODATA" | "SKIPDATA"

  // v11 ネクストホップフィールド (オプショナル)
  next-hop-interface-id: STRING?
  next-hop-subnet-id: STRING?
  next-hop-az-id: STRING?
  next-hop-vpc-id: STRING?
  next-hop-interface-type: STRING?  // "nat_gateway" | "regional_nat_gateway" | "vpc-endpoint" | ...

  // v11 タグフィールド (オプショナル)
  instance-tag: STRING?
  interface-tag: STRING?

  // v11 interface-type フィールド (オプショナル)
  interface-type: STRING?  // "amazon-elb" | "nat_gateway" | "efa" | ...

  // CloudWatch 自動発見フィールド
  @timestamp: TIMESTAMP
  @message: STRING
  @logStream: STRING
  @log: STRING
  @ingestionTime: TIMESTAMP
}
```

**注意:** v11 新フィールドはカスタムフォーマットで明示的に有効化した場合のみ記録される。有効化されていない場合はフィールドが存在しないため、`ispresent()` によるチェックが必須。

---

## Error Handling

### ドキュメント利用時の注意事項セクション

チートシートの末尾に以下の注意事項を含める:

| 問題 | 原因 | 対策 |
|---|---|---|
| クエリが0件を返す | v11 フィールドがカスタムフォーマットで有効化されていない | VPC Flow Log のカスタムフォーマット設定でフィールドを追加する |
| クエリが0件を返す | `ispresent()` なしでオプショナルフィールドをフィルタ | `ispresent(field)` をフィルタ前に追加する |
| 時刻フィルタが機能しない | ISO 8601 文字列で比較している | `toMillis(@timestamp)` とエポックミリ秒を使用する |
| `next-hop-az-id` と `az-id` が同じ値 | 同一AZ内通信 | クロスAZ検出には `!=` 条件を明示する |
| タグフィールドが空 | EC2/ENI にタグが付与されていない、またはフロログ設定でタグフィールドを未選択 | リソースへのタグ付与とフォーマット設定を確認する |

---

## Document Generation Rules

### Markdown フォーマット規則

1. **見出し階層**: H1（タイトル）→ H2（大セクション）→ H3（サブセクション）→ H4（個別クエリ例）
2. **コードブロック**: クエリ例はすべて ` ```sql ` ではなく ` ``` ` (言語指定なし、または `sql` でなく無指定) で囲む — Logs Insights QL は SQL ではないため
3. **テーブル**: フィールドリファレンスは Markdown テーブルを使用
4. **内部リンク**: 目次から各セクションへの GitHub 互換アンカーリンクを使用
5. **言語**: すべての説明文・見出し・コメントは日本語。フィールド名・コマンド名・関数名は英語のまま

### 各クエリ例の構成テンプレート

```markdown
#### クエリ例タイトル（日本語）

**シナリオ:** このクエリが必要な状況の説明

**期待される出力:** 出力カラムの説明

\```
(CloudWatch Logs Insights QL クエリ)
\```
```

---

## Correctness Properties

*プロパティはシステムのすべての有効な実行において真であるべき特性であり、人間が読める仕様と機械で検証可能な正確性保証の橋渡しとなる。*

### Property 1: フィールドリファレンス完全性

*For any* VPC Flow Logs フィールド（従来14フィールド、ネクストホップ5フィールド、タグ2フィールド、interface-type の計22フィールド）、当該フィールドはフィールドリファレンステーブルにフィールド名・データ型・日本語説明の3カラムで記載されていること。

**Validates: Requirements 2.1, 2.2, 2.3, 2.4, 2.5**

### Property 2: limit 句の普遍的存在

*For any* チートシート内のクエリコードブロック、当該クエリは値が50以上100以下の `limit` 句を含むこと。

**Validates: Requirements 7.2**

### Property 3: ispresent() によるオプショナルフィールド保護

*For any* チートシート内のクエリで v11 オプショナルフィールド（next-hop-interface-id, next-hop-subnet-id, next-hop-az-id, next-hop-vpc-id, next-hop-interface-type, instance-tag, interface-tag, interface-type）を `filter` または `stats ... by` で使用する場合、当該フィールドに対する `ispresent()` チェックがクエリ内でそれ以前に存在すること。

**Validates: Requirements 7.4**

### Property 4: Logs Insights QL 構文準拠

*For any* チートシート内のクエリコードブロック、当該クエリはパイプ区切りの CloudWatch Logs Insights QL 構文（fields, filter, parse, stats, sort, limit, display 等のコマンド）で記述されていること。OpenSearch PPL や SQL 構文は含まないこと。

**Validates: Requirements 7.1**

### Property 5: 時刻フィルタの正確な構文

*For any* チートシート内のクエリで時間ベースのフィルタリングを行う場合、`toMillis(@timestamp)` とエポックミリ秒を使用しており、ISO 8601 文字列比較を使用していないこと。

**Validates: Requirements 7.5**
