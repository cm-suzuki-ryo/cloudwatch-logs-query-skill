# スキル改善フィードバック

2026-06-10 動作確認（us-east-1）の結果に基づく改善提案。

---

## 1. `expand` コマンドの Logs Insights QL 対応を明記

**現状:** ナレッジベースでは `unnest`（Logs Insights QL）と `expand`（OpenSearch PPL）に分離して記載されている。

**事実:** Logs Insights QL でも `expand` コマンドが動作することを確認済み。記事（classmethod）でも Logs Insights QL として `expand` を使用。

**提案:** リファレンスの Commands テーブルまたは Tips に以下を追記：

```
- `expand` は Logs Insights QL でも利用可能（配列フィールドを複数レコードに展開）。`parse` の glob/regex で文字列として切り出した配列に対して動作する。`unnest` と同等。
```

---

## 2. `parse logfmt` のサンプルクエリを追加

**現状:** Tips に `logfmt` モードの存在は記載されているが、具体的なクエリ例がない。

**事実:** 構文は `parse @message logfmt as <alias>` で、`<alias>.key` のドット記法でフィールドにアクセスする。`parse logfmt @message` と書くと構文エラー。

**提案:** Sample Queries セクションに追加：

```
**Parse example (logfmt):**
parse @message logfmt as lf
| filter lf.service = "payment-service"
| display lf.service, lf.method, lf.path, lf.duration
```

---

## 3. `rate` 関数は正常動作する — ナレッジベースの記載を修正

**現状:** ナレッジベースの Time-series Functions セクションに「no working syntax could be confirmed in testing (all attempts returned compile errors); exact signature unverified」と記載。スキルの Tips にも「`rate` has no confirmed working syntax as of testing」と記載。

**事実（2026-06-10 us-east-1 実機検証）:**
```
fields toMillis(@timestamp) as ms
| stats rate(ms, 1m) as rateVal by bin(5m)
| limit 5
```
→ 正常実行。非ゼロの有意な値（~5000）を返す。`rate(field, period)` のシグネチャで動作確認。
定数フィールド（例: status=200のみ）に対しては `0` を返すが、これは変化率が0であることの正しい結果。

**提案:** リファレンス・スキル両方を以下に修正：

```markdown
| `rate(fieldName, period)` | Per-interval rate of change for a numeric field. Use with `stats ... by bin()`. Returns the average change per specified period within each time bin. |
```

Tips の該当箇所:
```
- `count_over_time`/`sum_over_time` pair with `stats ... by bin()`; `offset` shifts bin boundary alignment. `rate(field, period)` computes per-interval rate of change (returns 0 when field values are constant within the bin).
```

---

## 4. `histogram` 第2引数 — 公式ドキュメントの「バケット数」記載は不正確、実際はバケット幅

**現状:** 公式ドキュメントでは `histogram(fieldName: NumericLogField, buckets: number)` と記載し「Bucketizes numeric field values into the specified number of equal-width ranges」と説明。ナレッジベースでは `histogram(field, bucketWidth)` と正しく記載済み。

**事実（2026-06-10 us-east-1 実機検証）:**
- `histogram(toMillis(@timestamp), 5)` → 各バケットに3-4件（幅5ms）。36535件を5バケットに分割した場合の~7300件/バケットではない
- `histogram(toMillis(@timestamp), 60000)` → 各バケットに~1270-1290件、バケットラベルの差=60000（=1分幅）

→ 第2引数は明確に **バケット幅（bucket width）** である。公式ドキュメントの記載が不正確。

**提案:** ナレッジベースの現行記載は正しいため変更不要。ただしスキルの Tips に公式ドキュメントとの差異について注記を追加：

```
- `histogram(field, width)` は `by` 句で使うグルーピング関数（2nd arg はバケット幅）。公式ドキュメントでは「バケット数」と記載されているが、実機検証でバケット幅であることを確認済み
```

---

## 5. `expand` / `unnest` と `split` の組み合わせ制約を明記

**現状:** ナレッジベースに `expand` は「In practice works on arrays extracted as strings via parse glob/regex; auto-parsed JSON list types may not expand」と記載。`unnest` は「Flatten a list into multiple records」と記載。

**事実（2026-06-10 us-east-1 実機検証）:**
- `fields split(@message, " ") as parts | expand parts` → エラーなしだが配列はフラット展開されず、JSON配列文字列がそのまま1行で出力
- `fields split(@message, " ") as parts | unnest parts` → `MalformedQueryException`（構文エラー）

**提案:** ナレッジベースの `expand` / `unnest` 説明を以下に修正：

```markdown
| `expand` | Flatten a list/array field into multiple records. NOTE: `split()` の返り値に対しては構文エラーにはならないがフラット展開されない。`parse` glob/regex で抽出した文字列配列に対して動作する |
| `unnest` | Flatten a list into multiple records. NOTE: `split()` の返り値に対しては MalformedQueryException になる。`jsonParse` で取得したネイティブ配列に対して使用する |
```

---

## 6. `strcontains` 第3引数（case-insensitive）は依然として無効

**現状:** ナレッジベースに「the 3rd (case-insensitive) argument is accepted but had no effect in testing」と記載。公式ドキュメントでは「If the third parameter is set to true, the match is case-insensitive」と記載。

**事実（2026-06-10 us-east-1 実機検証）:**
```
fields strcontains(@message, "GET") as hasGet,
       strcontains(@message, "get") as hasGetLower,
       strcontains(@message, "get", true) as hasGetCI
```
→ `hasGet=1`, `hasGetLower=0`, `hasGetCI=0`

第3引数 `true` を指定しても case-insensitive にならない。公式ドキュメントの記載と実際の動作が一致していない。

**提案:** ナレッジベースの現行記載は正しいため変更不要。

---

## 7. `count_over_time(*)` は `count(*)` と完全同値

**事実（2026-06-10 us-east-1 実機検証）:**
```
stats count_over_time(*) as cot, count(*) as cnt by bin(1h)
```
→ 両者完全一致（12674, 23861）。`count_over_time(@message)` もフィールド名指定で動作し同じ結果。

**提案:** ナレッジベースの現行記載は正しい。

---

## 8. `jsonParse` のドットアクセスは別の `fields` コマンドに分ける必要がある

**事実（2026-06-10 us-east-1 実機検証）:**
```
# NG: 同じ fields 内でのドットアクセス → 出力されない
fields jsonParse('{"name":"test"}') as j, j.name as jName | limit 1
→ j のみ出力。jName は出力されない。

# OK: 2つの fields に分離 → 動作する
fields jsonParse(concat('{"ip":"', srcIp, '"}')) as j
| fields j.ip as jip
→ jip = "127.0.0.1" 正常出力
```

**提案:** ナレッジベースのJSON Functions セクションに以下を追記：

```
NOTE: `jsonParse` の結果に対するドットアクセス（`j.field`）は、`jsonParse` と **同一の `fields` コマンド内** では機能しない。2つの `fields` コマンドに分離する必要がある:
  fields jsonParse(@message) as j
  | fields j.status as statusCode, j.user as userName
```

---

## 9. `dedup` の後に `sort` は使えない

**事実（2026-06-10 us-east-1 実機検証）:**
```
# NG: dedup → sort
parse @message /.../ | dedup srcIp, status | sort @timestamp asc
→ MalformedQueryException: "unexpected symbol found sort"

# OK: sort → dedup
parse @message /.../ | sort @timestamp asc | dedup srcIp, status | limit 5
→ 正常動作
```

**提案:** ナレッジベースの `dedup` 説明に制約を追記：

```
| `dedup` | Remove duplicates. **NOTE: `dedup` の後に `sort` は使用不可。** ソートが必要な場合は `sort` → `dedup` の順序で記述する |
```

---

## 10. `addtotals` は `fields` の後では動作するが、`stats` の後では `Total` カラムが出ない

**事実（2026-06-10 us-east-1 実機検証）:**
```
# OK: fields の後 → Total カラムが追加される
parse @message /.../ | fields toNumber(status) as s1, toNumber(status) * 2 as s2
| addtotals | limit 3
→ s1=200, s2=400, Total=800 (status(200) + s1(200) + s2(400) = 800、全数値フィールド合算)

# NG: stats の後 → Total カラムが出力されない
stats count(*) as cnt, sum(toNumber(status)) as total by bin(1h)
| addtotals fieldname=RowSum
→ cnt, total のみ出力。RowSum なし。
```

**考察:** `stats ... by bin()` の結果に対して `addtotals` を適用すると、`by` 句のフィールド（bin値=タイムスタンプ）は数値ではないため合算対象外。残りの数値フィールド（cnt, total）のみが対象だが、API出力には Total/RowSum カラムが出現しなかった。

**提案:** ナレッジベースの `addtotals` 説明に以下を追記：

```
| `addtotals` | Add row-total column summing all numeric fields. Works after `fields` command. NOTE: `stats ... by` の結果に対しては Total カラムが出力されない場合がある（`by` フィールドは非数値扱い） |
```

---

## 11. `relevantfields` は `parse` で作成したフィールドでは動作しない

**事実（2026-06-10 us-east-1 実機検証）:**
```
# NG: parse で作ったフィールドを指定
parse @message /...(?<status>\d+)/ | relevantfields status where toNumber(status) = 200
→ "Query cannot be compiled"

# OK: @message など既存フィールドを指定
relevantfields @message where @message like /GET/
→ @fieldName=@message, @relevanceScore=1.0
```

**提案:** ナレッジベースの `relevantfields` 説明を更新：

```
| `relevantfields` | Surface fields most correlated with a condition. **`where` 句必須。** `parse` で作成したフィールドは指定不可。`@message` や自動検出フィールドのみ対応 |
```

---

## 12. `earliest` / `latest` の返り値は epoch ミリ秒

**事実（2026-06-10 us-east-1 実機検証）:**
```
stats earliest(@timestamp) as first, latest(@timestamp) as last
→ first=1780503582214, last=1780507082622 (epoch milliseconds)
```

タイムスタンプ形式ではなく数値で返る。`fromMillis()` でタイムスタンプに変換可能。

**提案:** ナレッジベースの Stats 関数に注記追加：

```
- `earliest(field)`, `latest(field)` — returns epoch milliseconds (not human-readable timestamp). Use `fromMillis()` to convert.
```

---

## 13. `diff` は `pattern` の後でないと使用不可

**事実（2026-06-10 us-east-1 実機検証）:**
```
# NG: filter の後に直接 diff
filter @message like /GET/ | diff
→ "Diff compilation Failed : Diff operation has to be used after Pattern"

# OK: pattern → diff
pattern @message | diff
```

**提案:** ナレッジベースの `diff` 説明を更新：

```
| `diff` | Compare current vs previous equal-length time period. **`pattern` の後でのみ使用可能** |
```

---

## 14. `limit any` はスキャン量を大幅に削減する（実測）

**事実（2026-06-10 us-east-1 実機検証）:**
```
fields @timestamp, @message | limit any 3
→ recordsScanned: 373 (全36535件中、わずか1%のスキャンで停止)

fields @timestamp, @message | limit 3
→ recordsScanned: 36535 (全件スキャン後に3件返す)
```

**提案:** ナレッジベース・スキルの現行記載は正しい。具体的なスキャン削減率を追記：

```
- `limit any N` は条件を満たす N 件が見つかった時点でスキャンを停止する。通常の `limit N` はフルスキャン後に N 件を切り出す。コスト削減に有効（実測: 36535件 → 373件スキャンで停止）
```

---

## 全関数動作サマリー（2026-06-10 us-east-1 実機検証）

### 正常動作確認（ドキュメント通り）
| カテゴリ | 関数 |
|----------|------|
| 数値 | abs, ceil, floor, greatest, least, log, round, sqrt, haversine, toNumber, toInt, toLong, toDouble |
| 日時 | bin, datefloor, dateceil, fromMillis, toMillis, now |
| 一般 | ispresent, coalesce, case, if |
| JSON | jsonParse, jsonStringify |
| IP | isValidIp, isValidIpV4, isValidIpV6, isIpInSubnet, isIpv4InSubnet, ipv4ToNumber, isPrivateIP, isPublicIP, isReservedIP |
| 文字列 | isempty, isblank, concat, ltrim, rtrim, trim, strlen, toupper, tolower, substr, replace, regex_replace, startsWith, endsWith, urlencode, urldecode, base64encode, base64decode, split |
| Stats | count, count(*), count_distinct, sum, avg, min, max, pct, stddev, values, earliest, latest |
| ハッシュ | md5, sha256 |
| 時系列 | rate, count_over_time, sum_over_time, histogram, offset |
| コマンド | fields, filter, parse, sort, limit, limit any, dedup, pattern, display, addtotals, relevantfields |

### ドキュメントとの差異・制約
| 項目 | ドキュメント記載 | 実動作 |
|------|------------------|--------|
| `histogram` 第2引数 | バケット数 | **バケット幅** |
| `strcontains` 第3引数 | case-insensitive有効 | **無効（常にcase-sensitive）** |
| `rate` | シグネチャ記載あり | **動作する**（ナレッジベースの「動作しない」が誤り） |
| `jsonParse` + ドットアクセス | 直接アクセス可能 | **別 `fields` コマンドに分離必須** |
| `dedup` → `sort` | 記載なし | **構文エラー**（逆順は可） |
| `diff` | 時期比較 | **`pattern` の後でのみ使用可** |
| `addtotals` after `stats` | row-total追加 | **Total カラムが出力されない場合あり** |
| `relevantfields` | フィールド指定 | **`parse` フィールド不可、`@` フィールドのみ** |
| `earliest`/`latest` | 値を返す | **epoch ミリ秒で返る** |

### 未検証（テストデータ不足）
- `isIpv6InSubnet`（IPv6データなし）
- `anomaly`（十分な時系列データ必要）
- `parse XML`（XMLログなし）
- `join`, `subqueries`, `lookup`（複数ロググループ/ルックアップテーブル必要）
