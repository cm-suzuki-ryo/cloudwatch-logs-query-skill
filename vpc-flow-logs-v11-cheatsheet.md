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

| カスタムフォーマット名 | Logs Insights フィールド名 | データ型 | 説明 |
|---|---|---|---|
| version | version | INT | Flow Log バージョン |
| account-id | accountId | STRING | AWS アカウント ID |
| interface-id | interfaceId | STRING | ネットワークインターフェース ID |
| srcaddr | srcaddr | STRING | 送信元 IP アドレス |
| dstaddr | dstaddr | STRING | 宛先 IP アドレス |
| srcport | srcport | INT | 送信元ポート番号 |
| dstport | dstport | INT | 宛先ポート番号 |
| protocol | protocol | INT | IANA プロトコル番号 |
| packets | packets | LONG | パケット数 |
| bytes | bytes | LONG | バイト数 |
| start | start | LONG | キャプチャ開始時刻（UNIX タイム） |
| end | end | LONG | キャプチャ終了時刻（UNIX タイム） |
| action | action | STRING | ACCEPT または REJECT |
| log-status | logStatus | STRING | ログ記録ステータス（OK, NODATA, SKIPDATA） |

### v11 新規フィールド

#### ネクストホップ系フィールド（5フィールド）

| カスタムフォーマット名 | Logs Insights フィールド名 | データ型 | 説明 |
|---|---|---|---|
| next-hop-interface-id | nextHopInterfaceId | STRING | ネクストホップのネットワークインターフェース ID |
| next-hop-subnet-id | nextHopSubnetId | STRING | ネクストホップのサブネット ID |
| next-hop-az-id | nextHopAzId | STRING | ネクストホップのアベイラビリティゾーン ID |
| next-hop-vpc-id | nextHopVpcId | STRING | ネクストホップの VPC ID |
| next-hop-interface-type | nextHopInterfaceType | STRING | ネクストホップのインターフェースタイプ（nat_gateway, regional_nat_gateway, vpc-endpoint 等） |

#### AZ 識別フィールド（1フィールド）

| カスタムフォーマット名 | Logs Insights フィールド名 | データ型 | 説明 |
|---|---|---|---|
| az-id | azId | STRING | 送信元 ENI が属するアベイラビリティゾーン ID（例: apne1-az1）。クロスAZ検出クエリで nextHopAzId と比較するために使用。カスタムフォーマットに含める必要あり |

#### タグ系フィールド（2フィールド）

| カスタムフォーマット名 | Logs Insights フィールド名 | データ型 | 説明 |
|---|---|---|---|
| instance-tag | instanceTag | STRING | EC2 インスタンスに付与されたタグ値 |
| interface-tag | interfaceTag | STRING | ENI に付与されたタグ値 |

#### その他（1フィールド）

| カスタムフォーマット名 | Logs Insights フィールド名 | データ型 | 説明 |
|---|---|---|---|
| interface-type | interfaceType | STRING | ENI のタイプ（amazon-elb, nat_gateway, efa 等） |

---

## クロスAZ通信検出

AZ（アベイラビリティゾーン）間の通信が発生すると、AZ 間データ転送料金が発生します。nextHopAzId フィールドを使用することで、送信元 AZ とネクストホップ AZ が異なるフローを特定し、コスト発生源を可視化できます。

> **注意:** azId フィールドは v2 基本14フィールドには含まれません。クロスAZ検出クエリを使用するには、カスタムフォーマットに az-id と next-hop-az-id の両方を含める必要があります。

#### 基本的なクロスAZ通信検出

**シナリオ:** 送信元 AZ とネクストホップ AZ が異なるフローを一覧表示し、クロスAZ通信の存在を確認する。

**期待される出力:** タイムスタンプ、インターフェースID、送信元/宛先IP、送信元AZ、ネクストホップAZ、バイト数

```
filter ispresent(nextHopAzId) and nextHopAzId != "-"
| filter azId != nextHopAzId
| fields @timestamp, interfaceId, srcaddr, dstaddr, azId, nextHopAzId, bytes
| sort bytes desc
| limit 50
```

#### AZペア別トラフィック量集計

**シナリオ:** どの AZ ペア間で最も多くのデータ転送が発生しているかを把握し、コスト削減の優先順位を決定する。

**期待される出力:** 送信元AZ、ネクストホップAZ、合計バイト数、フロー数

```
filter ispresent(nextHopAzId) and nextHopAzId != "-"
| filter azId != nextHopAzId
| stats sum(bytes) as totalBytes, count(*) as flowCount by azId, nextHopAzId
| sort totalBytes desc
| limit 50
```

#### クロスAZ通信量上位インターフェース

**シナリオ:** クロスAZ通信を最も多く発生させているインターフェースを特定し、アーキテクチャ改善の対象を絞り込む。

**期待される出力:** ネクストホップインターフェースID、送信元AZ、ネクストホップAZ、合計バイト数

```
filter ispresent(nextHopAzId) and nextHopAzId != "-"
| filter ispresent(nextHopInterfaceId) and nextHopInterfaceId != "-"
| filter azId != nextHopAzId
| stats sum(bytes) as totalBytes by nextHopInterfaceId, azId, nextHopAzId
| sort totalBytes desc
| limit 50
```

---

## NAT Gateway ルーティング分析

NAT Gateway はデータ処理料金（GB あたり課金）が発生するため、どのリソースがどれだけのトラフィックを NAT Gateway 経由で送信しているかを把握することがコスト最適化に重要です。nextHopInterfaceType フィールドで NAT Gateway 経由のフローを特定できます。

**注意:** nextHopInterfaceType の NAT Gateway 関連の値はアンダースコア区切りです:
- `"nat_gateway"` — AZ 単位の NAT Gateway
- `"regional_nat_gateway"` — リージョナル NAT Gateway

#### NAT Gateway 経由トラフィック検出

**シナリオ:** NAT Gateway（AZ 単位およびリージョナル）を経由する全トラフィックを一覧表示し、利用状況を確認する。

**期待される出力:** タイムスタンプ、送信元/宛先IP、ネクストホップインターフェースID、ネクストホップインターフェースタイプ、バイト数、パケット数

```
filter ispresent(nextHopInterfaceType) and nextHopInterfaceType != "-"
| filter nextHopInterfaceType = "nat_gateway" or nextHopInterfaceType = "regional_nat_gateway"
| fields @timestamp, srcaddr, dstaddr, nextHopInterfaceId, nextHopInterfaceType, bytes, packets
| sort bytes desc
| limit 50
```

#### VPC/サブネット別 NAT Gateway トラフィック集計

**シナリオ:** どの VPC・サブネットから NAT Gateway への通信が多いかを集計し、コスト配分やアーキテクチャ見直しの判断材料とする。

**期待される出力:** ネクストホップVPC ID、ネクストホップサブネットID、インターフェースタイプ、合計バイト数、合計パケット数

```
filter ispresent(nextHopInterfaceType) and nextHopInterfaceType != "-"
| filter nextHopInterfaceType = "nat_gateway" or nextHopInterfaceType = "regional_nat_gateway"
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets by nextHopVpcId, nextHopSubnetId, nextHopInterfaceType
| sort totalBytes desc
| limit 50
```

#### NAT Gateway 経由トップ送信元

**シナリオ:** NAT Gateway を最も多く利用している送信元 IP アドレスを特定し、コスト削減対象のワークロードを絞り込む。

**期待される出力:** 送信元IPアドレス、合計バイト数、フロー数

```
filter ispresent(nextHopInterfaceType) and nextHopInterfaceType != "-"
| filter nextHopInterfaceType = "nat_gateway" or nextHopInterfaceType = "regional_nat_gateway"
| stats sum(bytes) as totalBytes, count(*) as flowCount by srcaddr
| sort totalBytes desc
| limit 50
```

---

## タグベース集計

EC2 インスタンスや ENI に付与されたタグを活用することで、アプリケーション単位やチーム単位でのネットワーク利用状況を可視化できます。instanceTag と interfaceTag フィールドにより、リソースの論理的なグループ単位でトラフィックを集計・分析できます。

#### instance-tag 別トラフィック集計

**シナリオ:** EC2 インスタンスに付与されたタグ（例: アプリケーション名、チーム名）ごとにトラフィック量を集計し、リソース利用状況を把握する。

**期待される出力:** インスタンスタグ値、合計バイト数、合計パケット数、フロー数

```
filter ispresent(instanceTag)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by instanceTag
| sort totalBytes desc
| limit 50
```

#### interface-tag 別トラフィック集計

**シナリオ:** ENI に付与されたタグごとにトラフィック量を集計し、ネットワークインターフェース単位での利用状況を把握する。

**期待される出力:** インターフェースタグ値、合計バイト数、合計パケット数、フロー数

```
filter ispresent(interfaceTag)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by interfaceTag
| sort totalBytes desc
| limit 50
```

#### タグ × action セキュリティ分析

**シナリオ:** タグ別に ACCEPT/REJECT の分布を分析し、特定のアプリケーションやチームのリソースで拒否されたトラフィックが多い場合にセキュリティグループの見直しを検討する。

**期待される出力:** インスタンスタグ値、アクション（ACCEPT/REJECT）、合計バイト数、フロー数

```
filter ispresent(instanceTag)
| stats sum(bytes) as totalBytes, count(*) as flowCount by instanceTag, action
| sort totalBytes desc
| limit 50
```

---

## interface-type 別トラフィック分類

interfaceType フィールドは ENI の種類を示し、トラフィックがどのような AWS サービス（ELB、NAT Gateway、EFA 等）に関連しているかを分類できます。nextHopInterfaceType と組み合わせることで、ソースからネクストホップへのエンドツーエンドのルーティング経路を分析できます。

#### interface-type 別トラフィック分布

**シナリオ:** ENI タイプごとのトラフィック分布を把握し、どのサービスタイプが最も多くのネットワークリソースを消費しているかを確認する。

**期待される出力:** インターフェースタイプ、合計バイト数、合計パケット数、フロー数

```
filter ispresent(interfaceType)
| stats sum(bytes) as totalBytes, sum(packets) as totalPackets, count(*) as flowCount by interfaceType
| sort totalBytes desc
| limit 50
```

#### ソース → ネクストホップ ルーティングパス分析

**シナリオ:** 送信元のインターフェースタイプとネクストホップのインターフェースタイプの組み合わせを分析し、トラフィックがどのような経路でルーティングされているかを把握する。

**期待される出力:** ソースインターフェースタイプ、ネクストホップインターフェースタイプ、合計バイト数、フロー数

```
filter ispresent(interfaceType) and ispresent(nextHopInterfaceType) and nextHopInterfaceType != "-"
| stats sum(bytes) as totalBytes, count(*) as flowCount by interfaceType, nextHopInterfaceType
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

### フィールド名の変換規則

VPC Flow Logs のフィールド名は、カスタムフォーマットではハイフン区切り（例: next-hop-az-id）ですが、CloudWatch Logs Insights ではキャメルケース（例: nextHopAzId）で参照します。バッククォート不要です。

| カスタムフォーマット指定名 | Logs Insights でのフィールド名 |
|---|---|
| next-hop-az-id | nextHopAzId |
| next-hop-interface-id | nextHopInterfaceId |
| next-hop-interface-type | nextHopInterfaceType |
| next-hop-subnet-id | nextHopSubnetId |
| next-hop-vpc-id | nextHopVpcId |
| instance-tag | instanceTag |
| interface-tag | interfaceTag |
| interface-type | interfaceType |
| az-id | azId |

### トラブルシューティング

| 問題 | 原因 | 対策 |
|---|---|---|
| クエリが 0 件を返す | v11 フィールドがカスタムフォーマットで有効化されていない | VPC Flow Log のカスタムフォーマット設定で対象フィールドを追加する |
| クエリが 0 件を返す | バッククォート付きハイフン区切り名（例: \`next-hop-az-id\`）を使用している | キャメルケース名（例: nextHopAzId）に変更する。バッククォートは不要 |
| クエリが 0 件を返す | `ispresent()` なしでオプショナルフィールドをフィルタしている | `ispresent(field)` をフィルタ条件の前に追加する |
| 時刻フィルタが機能しない | ISO 8601 文字列で比較している | `toMillis(@timestamp)` とエポックミリ秒を使用する |
| nextHopAzId と azId が同じ値 | 同一 AZ 内通信 | クロスAZ検出には `!=` 条件を明示する |
| タグフィールドが空 | EC2/ENI にタグが付与されていない、またはフローログ設定でタグフィールドを未選択 | リソースへのタグ付与とカスタムフォーマット設定を確認する |
| タグの値が期待と一致しない | タグ値がパーセントエンコードされている | デコード後の値で比較する（下記「タグフィールドのパーセントエンコーディング」参照） |
| next-hop-* フィールドが `"-"` | 対応する通信経路が存在しない（ローカル配信等） | `ispresent()` に加えて `!= "-"` でフィルタする |

### タグフィールドのパーセントエンコーディング

instanceTag および interfaceTag フィールドの値は **パーセントエンコード** された状態で記録されます。クエリでタグ値を条件に使う場合は、エンコード後の文字列で比較する必要があります。

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
filter ispresent(instanceTag)
| filter instanceTag like /my%2Dapp/
```

### next-hop-* フィールドの "-" 値について

nextHopInterfaceId、nextHopSubnetId、nextHopAzId、nextHopVpcId、nextHopInterfaceType の各フィールドは、対応するネクストホップ情報が存在しない場合（ローカル配信、同一サブネット内通信等）に `"-"` が記録されます。

`ispresent()` は `"-"` が格納されているレコードも true を返すため、有効なネクストホップ情報のみを対象にする場合は `!= "-"` 条件を併用してください。

```
# 推奨パターン
filter ispresent(nextHopAzId) and nextHopAzId != "-"
```

### クエリ構文の注意点

- **`limit` 必須:** すべてのクエリに `limit 50` 〜 `limit 100` を含めてください
- **`ispresent()` 使用:** v11 オプショナルフィールドでフィルタ・集計する前に必ず `ispresent()` で存在確認を行ってください
- **時刻フィルタ:** クエリ内で時間絞り込みを行う場合は `toMillis(@timestamp)` とエポックミリ秒を使用してください（ISO 8601 文字列比較は結果が 0 件になります）
- **コンソール時間範囲:** 最も効果的なスキャン削減は、コンソール UI または API の `startTime`/`endTime` で時間範囲を狭めることです
