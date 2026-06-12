# VPC Flow Logs v11 CloudWatch Logs Insights クエリチートシート

## はじめに

本ドキュメントは、VPC Flow Logs バージョン11 で追加されたフィールド（ネクストホップ系5フィールド、タグ系2フィールド、interface-type）を活用した CloudWatch Logs Insights クエリパターンを、ユースケース別に整理した日本語リファレンスです。

**対象読者:** AWS 運用エンジニア、ネットワークエンジニア、コスト最適化担当者

**前提条件:**
- VPC Flow Logs がカスタムフォーマットで有効化されていること
- v11 新フィールドがカスタムフォーマットに含まれていること
- CloudWatch Logs Insights コンソールまたは API でクエリを実行できる環境があること

---

## 目次

- [フィールドリファレンス](#フィールドリファレンス)
  - [従来フィールド（v2 基本14フィールド）](#従来フィールドv2-基本14フィールド)
  - [v11 新規フィールド](#v11-新規フィールド)
- [クロスAZ通信検出](#クロスaz通信検出)
- [NAT Gateway ルーティング分析](#nat-gateway-ルーティング分析)
- [タグベース集計](#タグベース集計)
- [interface-type 別トラフィック分類](#interface-type-別トラフィック分類)
- [基本パターン（従来フィールドのみ）](#基本パターン従来フィールドのみ)
- [補足・注意事項](#補足注意事項)

---

## フィールドリファレンス

### 従来フィールド（v2 基本14フィールド）

| フィールド名 | データ型 | 説明 |
|---|---|---|
| version | INT | Flow Log バージョン |
| account-id | STRING | AWS アカウント ID |
| interface-id | STRING | ネットワークインターフェース ID |
| srcaddr | STRING | 送信元 IP アドレス |
| dstaddr | STRING | 宛先 IP アドレス |
| srcport | INT | 送信元ポート番号 |
| dstport | INT | 宛先ポート番号 |
| protocol | INT | IANA プロトコル番号 |
| packets | LONG | パケット数 |
| bytes | LONG | バイト数 |
| start | LONG | キャプチャ開始時刻（UNIX タイム） |
| end | LONG | キャプチャ終了時刻（UNIX タイム） |
| action | STRING | ACCEPT または REJECT |
| log-status | STRING | ログ記録ステータス（OK, NODATA, SKIPDATA） |

### v11 新規フィールド

#### ネクストホップ系フィールド（5フィールド）

| フィールド名 | データ型 | 説明 |
|---|---|---|
| next-hop-interface-id | STRING | ネクストホップのネットワークインターフェース ID |
| next-hop-subnet-id | STRING | ネクストホップのサブネット ID |
| next-hop-az-id | STRING | ネクストホップのアベイラビリティゾーン ID |
| next-hop-vpc-id | STRING | ネクストホップの VPC ID |
| next-hop-interface-type | STRING | ネクストホップのインターフェースタイプ（nat_gateway, regional_nat_gateway, vpc-endpoint 等） |

#### AZ 識別フィールド（1フィールド）

| フィールド名 | データ型 | 説明 |
|---|---|---|
| az-id | STRING | 送信元 ENI が属するアベイラビリティゾーン ID（例: apne1-az1）。クロスAZ検出クエリで `next-hop-az-id` と比較するために使用。カスタムフォーマットに含める必要あり |

#### タグ系フィールド（2フィールド）

| フィールド名 | データ型 | 説明 |
|---|---|---|
| instance-tag | STRING | EC2 インスタンスに付与されたタグ値 |
| interface-tag | STRING | ENI に付与されたタグ値 |

#### その他（1フィールド）

| フィールド名 | データ型 | 説明 |
|---|---|---|
| interface-type | STRING | ENI のタイプ（amazon-elb, nat_gateway, efa 等） |

---

## クロスAZ通信検出

AZ（アベイラビリティゾーン）間の通信が発生すると、AZ 間データ転送料金が発生します。`next-hop-az-id` フィールドを使用することで、送信元 AZ とネクストホップ AZ が異なるフローを特定し、コスト発生源を可視化できます。

> **注意:** `az-id` フィールドは v2 基本14フィールドには含まれません。クロスAZ検出クエリを使用するには、カスタムフォーマットに `az-id` と `next-hop-az-id` の両方を含める必要があります。

#### 基本的なクロスAZ通信検出

**シナリオ:** 送信元 AZ とネクストホップ AZ が異なるフローを一覧表示し、クロスAZ通信の存在を確認する。

**期待される出力:** タイムスタンプ、インターフェースID、送信元/宛先IP、送信元AZ、ネクストホップAZ、バイト数

```
filter ispresent(`next-hop-az-id`) and `next-hop-az-id` != "-"
| filter `az-id` != `next-hop-az-id`
| fields @timestamp, `interface-id`, `srcaddr`, `dstaddr`, `az-id`, `next-hop-az-id`, bytes
| sort bytes desc
| limit 50
```

#### AZペア別トラフィック量集計

**シナリオ:** どの AZ ペア間で最も多くのデータ転送が発生しているかを把握し、コスト削減の優先順位を決定する。

**期待される出力:** 送信元AZ、ネクストホップAZ、合計バイト数、フロー数

```
filter ispresent(`next-hop-az-id`) and `next-hop-az-id` != "-"
| filter `az-id` != `next-hop-az-id`
| stats sum(bytes) as totalBytes, count(*) as flowCount by `az-id`, `next-hop-az-id`
| sort totalBytes desc
| limit 50
```

#### クロスAZ通信量上位インターフェース

**シナリオ:** クロスAZ通信を最も多く発生させているインターフェースを特定し、アーキテクチャ改善の対象を絞り込む。

**期待される出力:** ネクストホップインターフェースID、送信元AZ、ネクストホップAZ、合計バイト数

```
filter ispresent(`next-hop-az-id`) and `next-hop-az-id` != "-"
| filter ispresent(`next-hop-interface-id`) and `next-hop-interface-id` != "-"
| filter `az-id` != `next-hop-az-id`
| stats sum(bytes) as totalBytes by `next-hop-interface-id`, `az-id`, `next-hop-az-id`
| sort totalBytes desc
| limit 50
```

---

## NAT Gateway ルーティング分析

NAT Gateway はデータ処理料金（GB あたり課金）が発生するため、どのリソースがどれだけのトラフィックを NAT Gateway 経由で送信しているかを把握することがコスト最適化に重要です。`next-hop-interface-type` フィールドで NAT Gateway 経由のフローを特定できます。

**注意:** `next-hop-interface-type` の NAT Gateway 関連の値はアンダースコア区切りです:
- `"nat_gateway"` — AZ 単位の NAT Gateway
- `"regional_nat_gateway"` — リージョナル NAT Gateway

#### NAT Gateway 経由トラフィック検出

**シナリオ:** NAT Gateway（AZ 単位およびリージョナル）を経由する全トラフィックを一覧表示し、利用状況を確認する。

**期待される出力:** タイムスタンプ、送信元/宛先IP、ネクストホップインターフェースID、ネクストホップインターフェースタイプ、バイト数、パケット数

```
filter ispresent(`next-hop-interface-type`) and `next-hop-interface-type` != "-"
| filter `next-hop-interface-type` = "nat_gateway" or `next-hop-interface-type` = "regional_nat_gateway"
| fields @timestamp, `srcaddr`, `dstaddr`, `next-hop-interface-id`, `next-hop-interface-type`, bytes, packets
| sort bytes desc
| limit 50
```

#### VPC/サブネット別 NAT Gateway トラフィック集計

**シナリオ:** どの VPC・サブネットから NAT Gateway への通信が多いかを集計し、コスト配分やアーキテクチャ見直しの判断材料とする。

**期待される出力:** ネクストホップVPC ID、ネクストホップサブネットID、インターフェースタイプ、合計バイト数、合計パケット数

```
filter ispresent(`next-hop-interface-type`) and `next-hop-interface-type` != "-"
| filter `next-hop-interface-type` = "nat_gateway" or `next-hop-interface-type` = "regional_nat_gateway"
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets by `next-hop-vpc-id`, `next-hop-subnet-id`, `next-hop-interface-type`
| sort totalBytes desc
| limit 50
```

#### NAT Gateway 経由トップ送信元

**シナリオ:** NAT Gateway を最も多く利用している送信元 IP アドレスを特定し、コスト削減対象のワークロードを絞り込む。

**期待される出力:** 送信元IPアドレス、合計バイト数、フロー数

```
filter ispresent(`next-hop-interface-type`) and `next-hop-interface-type` != "-"
| filter `next-hop-interface-type` = "nat_gateway" or `next-hop-interface-type` = "regional_nat_gateway"
| stats sum(bytes) as totalBytes, count(*) as flowCount by srcaddr
| sort totalBytes desc
| limit 50
```

---

## タグベース集計

EC2 インスタンスや ENI に付与されたタグを活用することで、アプリケーション単位やチーム単位でのネットワーク利用状況を可視化できます。`instance-tag` と `interface-tag` フィールドにより、リソースの論理的なグループ単位でトラフィックを集計・分析できます。

#### instance-tag 別トラフィック集計

**シナリオ:** EC2 インスタンスに付与されたタグ（例: アプリケーション名、チーム名）ごとにトラフィック量を集計し、リソース利用状況を把握する。

**期待される出力:** インスタンスタグ値、合計バイト数、合計パケット数、フロー数

```
filter ispresent(`instance-tag`)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by `instance-tag`
| sort totalBytes desc
| limit 50
```

#### interface-tag 別トラフィック集計

**シナリオ:** ENI に付与されたタグごとにトラフィック量を集計し、ネットワークインターフェース単位での利用状況を把握する。

**期待される出力:** インターフェースタグ値、合計バイト数、合計パケット数、フロー数

```
filter ispresent(`interface-tag`)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by `interface-tag`
| sort totalBytes desc
| limit 50
```

#### タグ × action セキュリティ分析

**シナリオ:** タグ別に ACCEPT/REJECT の分布を分析し、特定のアプリケーションやチームのリソースで拒否されたトラフィックが多い場合にセキュリティグループの見直しを検討する。

**期待される出力:** インスタンスタグ値、アクション（ACCEPT/REJECT）、合計バイト数、フロー数

```
filter ispresent(`instance-tag`)
| stats sum(bytes) as totalBytes, count(*) as flowCount by `instance-tag`, action
| sort totalBytes desc
| limit 50
```

---

## interface-type 別トラフィック分類

`interface-type` フィールドは ENI の種類を示し、トラフィックがどのような AWS サービス（ELB、NAT Gateway、EFA 等）に関連しているかを分類できます。`next-hop-interface-type` と組み合わせることで、ソースからネクストホップへのエンドツーエンドのルーティング経路を分析できます。

#### interface-type 別トラフィック分布

**シナリオ:** ENI タイプごとのトラフィック分布を把握し、どのサービスタイプが最も多くのネットワークリソースを消費しているかを確認する。

**期待される出力:** インターフェースタイプ、合計バイト数、合計パケット数、フロー数

```
filter ispresent(`interface-type`)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by `interface-type`
| sort totalBytes desc
| limit 50
```

#### ソース → ネクストホップ ルーティングパス分析

**シナリオ:** 送信元のインターフェースタイプとネクストホップのインターフェースタイプの組み合わせを分析し、トラフィックがどのような経路でルーティングされているかを把握する。

**期待される出力:** ソースインターフェースタイプ、ネクストホップインターフェースタイプ、合計バイト数、フロー数

```
filter ispresent(`interface-type`) and ispresent(`next-hop-interface-type`) and `next-hop-interface-type` != "-"
| stats sum(bytes) as totalBytes, count(*) as flowCount by `interface-type`, `next-hop-interface-type`
| sort totalBytes desc
| limit 50
```

---

## 基本パターン（従来フィールドのみ）

以下のクエリは v11 新フィールドを使用せず、従来の v2 基本フィールドのみで実行できる一般的な分析パターンです。

#### Top Talkers 分析

**シナリオ:** 最も多くのデータを送受信している IP アドレスペアを特定し、ネットワーク帯域の主な消費者を把握する。

**期待される出力:** 送信元IPアドレス、宛先IPアドレス、合計バイト数

```
stats sum(bytes) as totalBytes by srcaddr, dstaddr
| sort totalBytes desc
| limit 50
```

#### REJECT アクション分析

**シナリオ:** セキュリティグループまたはネットワーク ACL により拒否されたトラフィックを分析し、不正アクセスの試行や設定ミスを検出する。

**期待される出力:** 送信元IPアドレス、宛先IPアドレス、宛先ポート番号、拒否回数

```
filter action = "REJECT"
| stats count(*) as rejectCount by srcaddr, dstaddr, dstport
| sort rejectCount desc
| limit 50
```

---

## 補足・注意事項

### トラブルシューティング

| 問題 | 原因 | 対策 |
|---|---|---|
| クエリが 0 件を返す | v11 フィールドがカスタムフォーマットで有効化されていない | VPC Flow Log のカスタムフォーマット設定で対象フィールドを追加する |
| クエリが 0 件を返す | `ispresent()` なしでオプショナルフィールドをフィルタしている | `ispresent(field)` をフィルタ条件の前に追加する |
| 時刻フィルタが機能しない | ISO 8601 文字列で比較している | `toMillis(@timestamp)` とエポックミリ秒を使用する |
| `next-hop-az-id` と `az-id` が同じ値 | 同一 AZ 内通信 | クロスAZ検出には `!=` 条件を明示する |
| タグフィールドが空 | EC2/ENI にタグが付与されていない、またはフローログ設定でタグフィールドを未選択 | リソースへのタグ付与とカスタムフォーマット設定を確認する |
| タグの値が期待と一致しない | タグ値がパーセントエンコードされている | デコード後の値で比較する（下記「タグフィールドのパーセントエンコーディング」参照） |
| next-hop-* フィールドが `"-"` | 対応する通信経路が存在しない（ローカル配信等） | `ispresent()` に加えて `!= "-"` でフィルタする |

### タグフィールドのパーセントエンコーディング

`instance-tag` および `interface-tag` フィールドの値は **パーセントエンコード** された状態で記録されます。クエリでタグ値を条件に使う場合は、エンコード後の文字列で比較する必要があります。

**主なエンコード例:**

| 元の文字 | エンコード後 |
|---|---|
| `-`（ハイフン） | `%2D` |
| ` `（スペース） | `%20` |
| `:`（コロン） | `%3A` |
| `/`（スラッシュ） | `%2F` |
| `=`（イコール） | `%3D` |

**例:** タグ値が `my-app` の場合、フローログには `my%2Dapp` として記録されます。

```
filter ispresent(`instance-tag`)
| filter `instance-tag` like /my%2Dapp/
```

### next-hop-* フィールドの "-" 値について

`next-hop-interface-id`、`next-hop-subnet-id`、`next-hop-az-id`、`next-hop-vpc-id`、`next-hop-interface-type` の各フィールドは、対応するネクストホップ情報が存在しない場合（ローカル配信、同一サブネット内通信等）に `"-"` が記録されます。

`ispresent()` は `"-"` が格納されているレコードも true を返すため、有効なネクストホップ情報のみを対象にする場合は `!= "-"` 条件を併用してください。

```
# 推奨パターン
filter ispresent(`next-hop-az-id`) and `next-hop-az-id` != "-"
```

### クエリ構文の注意点

- **`limit` 必須:** すべてのクエリに `limit 50` 〜 `limit 100` を含めてください
- **`ispresent()` 使用:** v11 オプショナルフィールドでフィルタ・集計する前に必ず `ispresent()` で存在確認を行ってください
- **時刻フィルタ:** クエリ内で時間絞り込みを行う場合は `toMillis(@timestamp)` とエポックミリ秒を使用してください（ISO 8601 文字列比較は結果が 0 件になります）
- **コンソール時間範囲:** 最も効果的なスキャン削減は、コンソール UI または API の `startTime`/`endTime` で時間範囲を狭めることです
