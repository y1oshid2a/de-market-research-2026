# SQLアンチパターン完全ガイド — データエンジニア向け

> データマート・テーブル設計・パフォーマンスチューニングで陥りやすい落とし穴とその対策

---

## 目次

1. [クエリのアンチパターン](#1-クエリのアンチパターン)
2. [テーブル設計のアンチパターン](#2-テーブル設計のアンチパターン)
3. [データマート・DWH設計のアンチパターン](#3-データマートdwh設計のアンチパターン)
4. [インデックス設計のアンチパターン](#4-インデックス設計のアンチパターン)
5. [パーティショニング・クラスタリングのアンチパターン](#5-パーティショニングクラスタリングのアンチパターン)
6. [クラウドDWH別のアンチパターン](#6-クラウドdwh別のアンチパターン)
7. [NULLの扱いに関するアンチパターン](#7-nullの扱いに関するアンチパターン)
8. [チェックリスト](#8-チェックリスト)

---

## 1. クエリのアンチパターン

### 1.1 `SELECT *` の使用

**アンチパターン:**
```sql
SELECT * FROM orders;
```

**問題点:**
- 不要な列を全てスキャンし、I/O・ネットワーク転送コストが増大する
- BigQueryなど列指向DWHでは**スキャンバイト数＝コスト**に直結する
- テーブルスキーマ変更時に意図しない副作用が生じる
- カバリングインデックスが利用できなくなる

**改善策:**
```sql
SELECT order_id, user_id, amount, created_at FROM orders;
```

> **注意（BigQuery）:** `SELECT * LIMIT 1000` としてもLIMITはコスト削減にならない。全列の全データをスキャンしてからLIMITを適用するため。

---

### 1.2 インデックス列に関数を適用する

**アンチパターン:**
```sql
-- indexed_at にインデックスがある場合でも使われない
WHERE DATE(indexed_at) = '2024-01-01'
WHERE UPPER(email) = 'EXAMPLE@DOMAIN.COM'
WHERE YEAR(created_at) = 2024
```

**問題点:**
- 列を関数でラップするとインデックスが無効化され、フルスキャンになる
- SQL ServerではCONVERT_IMPLICIT警告が出ることもある

**改善策:**
```sql
-- 範囲条件で書き直す
WHERE indexed_at >= '2024-01-01' AND indexed_at < '2024-01-02'
WHERE email = 'example@domain.com'  -- アプリ側でlowercase正規化
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
```

参考: [Use The Index, Luke — 関数とインデックス](https://use-the-index-luke.com/)

---

### 1.3 暗黙の型変換 (Implicit Type Conversion)

**アンチパターン:**
```sql
-- user_id が INT 型なのに文字列で比較
WHERE user_id = '12345'

-- created_at が TIMESTAMP 型なのに文字列で比較
WHERE created_at = '2024-01-01'
```

**問題点:**
- DBエンジンが自動的に型変換を行い、インデックスが利用できなくなる
- SQL Serverの実測値では、暗黙変換が発生するとロジカルリード数が**約10倍**増加するケースがある
- 変換ルールはDBMS間で異なり、移行時に予期せぬ不一致が起きる

**改善策:**
```sql
WHERE user_id = 12345          -- 数値型は数値リテラルで
WHERE created_at = TIMESTAMP '2024-01-01 00:00:00'  -- 明示的キャスト
```

参考: [How (not) to kill your SQL Server performance with implicit conversion - Data Mozart](https://data-mozart.com/how-not-to-kill-your-sql-server-performance-with-implicit-conversion/)

---

### 1.4 不要なサブクエリ・相関サブクエリ

**アンチパターン:**
```sql
-- 行ごとにサブクエリが実行される（相関サブクエリ）
SELECT
  user_id,
  (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.user_id) AS order_count
FROM users u;
```

**問題点:**
- 外側クエリの行数分だけサブクエリが実行される（O(n)の重複実行）
- 大規模テーブルでは致命的なパフォーマンス劣化

**改善策:**
```sql
-- JOINまたはウィンドウ関数で書き直す
SELECT
  u.user_id,
  COUNT(o.order_id) AS order_count
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id;
```

---

### 1.5 セルフジョイン（Self-Join）の乱用

**アンチパターン:**
```sql
-- 自己結合で前日比を計算しようとする
SELECT
  a.date,
  a.revenue,
  b.revenue AS prev_revenue
FROM daily_sales a
JOIN daily_sales b ON a.date = b.date + INTERVAL 1 DAY;
```

**問題点:**
- 同一テーブルを2回スキャンする
- ウィンドウ関数で代替可能なケースが多い

**改善策:**
```sql
SELECT
  date,
  revenue,
  LAG(revenue) OVER (ORDER BY date) AS prev_revenue
FROM daily_sales;
```

参考: [SQL Anti-Patterns for BigQuery - Towards Data Science](https://towardsdatascience.com/bigquery-anti-patterns-dacb61f8a3f/)

---

### 1.6 クロスジョイン（Cross Join）の意図しない使用

**アンチパターン:**
```sql
-- JOIN条件の書き忘れによる意図しないクロスジョイン
SELECT * FROM orders, users;

-- または明示的だが不要なクロスジョイン
SELECT * FROM orders CROSS JOIN users;
```

**問題点:**
- 結果行数はM×Nになる（100万行×100万行 = 1兆行）
- DWHでは莫大なスキャンコストが発生

**改善策:**
- 常に明示的なJOIN条件を記述する
- クロスジョインが必要な場合は、先にフィルタリングしてから結合する

---

### 1.7 HAVING句の誤用

**アンチパターン:**
```sql
-- WHERE で絞り込めるものを HAVING に書く
SELECT user_id, COUNT(*) AS cnt
FROM orders
GROUP BY user_id
HAVING user_id > 1000;  -- ← これは WHERE に書くべき
```

**問題点:**
- GROUP BY後のフィルタリングになり、不要な集計処理が発生する
- インデックスが活用されない

**改善策:**
```sql
SELECT user_id, COUNT(*) AS cnt
FROM orders
WHERE user_id > 1000    -- 集計前に絞り込む
GROUP BY user_id;
```

---

### 1.8 OR条件によるインデックス無効化

**アンチパターン:**
```sql
-- どちらかがインデックスを無効化する場合がある
WHERE status = 'active' OR created_at > '2024-01-01'
```

**問題点:**
- DBMSがOR条件の両辺を評価するために、インデックスを使えないことがある

**改善策:**
```sql
-- UNION ALL で分割する
SELECT * FROM orders WHERE status = 'active'
UNION ALL
SELECT * FROM orders WHERE created_at > '2024-01-01' AND status != 'active';
```

---

### 1.9 ワイルドカードの先頭一致（LIKE '%xxx'）

**アンチパターン:**
```sql
WHERE name LIKE '%山田'   -- 前方一致でないためインデックス不使用
WHERE email LIKE '%@gmail.com'
```

**問題点:**
- 先頭にワイルドカードがあるとインデックスが使えず、フルスキャンになる

**改善策:**
- 全文検索（Full-Text Search）機能を検討する
- 逆順インデックスや転置インデックスを活用する
- 後方一致の要件がある場合は設計を見直す

---

## 2. テーブル設計のアンチパターン

### 2.1 ジェイウォーク（信号無視）— 1カラムに複数値を格納

**アンチパターン:**
```sql
CREATE TABLE products (
  product_id INT,
  tags VARCHAR(255)  -- 'sports,outdoor,sale' と格納
);
```

**問題点:**
- 特定タグでの検索に`LIKE '%sports%'`が必要になりインデックス不使用
- タグの追加・削除が困難
- 集計クエリが極めて複雑になる

**改善策:**
```sql
-- 正規化：中間テーブルを使う
CREATE TABLE product_tags (
  product_id INT,
  tag VARCHAR(50)
);
```

参考: [SQLアンチパターン「ジェイウォーク」との付き合い方 - クリーヴァ株式会社](https://note.com/creava/n/n0afb33a99728)

---

### 2.2 メタデータトリブル — テーブルの分割しすぎ（テーブルシャーディング）

**アンチパターン:**
```sql
-- 月次データを別テーブルに分割
orders_2024_01
orders_2024_02
orders_2024_03
...
```

**問題点:**
- 跨がったクエリには大量のUNION ALLが必要になる
- DBMSがすべてのテーブルのメタデータとパーミッションを管理するためオーバーヘッドが大きい
- スキーマ変更時に全テーブルに適用が必要

**改善策:**
- テーブルパーティショニングを使う（1テーブル + パーティション定義）
- BigQueryでは`[PREFIX]_YYYYMMDD`のシャーディングは**アンチパターン**とされており、日付パーティションの利用が推奨される

参考: [BigQuery Query Performance Optimization Guide 2025 - e6data](https://www.e6data.com/query-and-cost-optimization-hub/how-to-optimize-bigquery-query-performance)

---

### 2.3 EAVモデル（Entity-Attribute-Value）の乱用

**アンチパターン:**
```sql
CREATE TABLE attributes (
  entity_id INT,
  attribute_name VARCHAR(100),
  attribute_value VARCHAR(500)
);
-- 例: (1, 'price', '1000'), (1, 'color', 'red')
```

**問題点:**
- 特定属性の取得にPIVOT/条件集計が必要になりクエリが複雑化
- 型安全性がなくなる（すべてVARCHAR）
- インデックスが効きにくく、大量データでのパフォーマンスが劣悪

**改善策:**
- 列挙可能な属性は通常の列として定義する
- 動的な属性はJSON型（PostgreSQL/MySQL/BigQueryのSTRUCT等）を検討する

---

### 2.4 過度な正規化（OLAPへの第3正規形適用）

**アンチパターン:**
```
-- OLTP設計をそのままDWH/データマートに使う
-- 10テーブル以上のJOINが必要なクエリが常態化
```

**問題点:**
- 分析クエリは広範なデータを読む傾向があり、多数のJOINはコスト増
- 列指向DWHはJOIN削減・非正規化の恩恵が大きい
- BigQuery/Redshift/Snowflakeはスタースキーマ・非正規化モデルに最適化されている

**改善策:**
- DWHレイヤーは正規化（Data Vault, 3NF）、データマートレイヤーは非正規化（スタースキーマ）という2層構造にする

参考:
- [データモデリングにおける非正規化とスタースキーマ - FLINTERS BASE BLOG](https://blog.flinters-base.co.jp/entry/2022/09/28/000000)
- [Star Schema Guide: Data Warehouse Modeling Explained - MotherDuck](https://motherduck.com/learn/star-schema-data-warehouse-guide/)

---

### 2.5 ファクトテーブルのグレイン不統一

**アンチパターン:**
```sql
-- 1行が「注文」なのか「注文明細」なのか曖昧なファクトテーブル
CREATE TABLE fact_orders (
  order_id INT,
  item_id INT,        -- 明細レベル
  order_total DECIMAL -- 注文レベル（重複集計の原因）
);
```

**問題点:**
- グレイン（1行の意味する粒度）が混在すると二重計上が発生する
- `SUM(order_total)`が商品数分重複加算される

**改善策:**
- ファクトテーブルのグレインを明確に定義し、一貫させる
- 複数粒度が必要なら別テーブルに分ける

---

### 2.6 NULL許容の過剰使用

**アンチパターン:**
```sql
CREATE TABLE users (
  user_id INT,
  name VARCHAR(100) NULL,   -- 必須項目のはずがNULL許容
  email VARCHAR(200) NULL
);
```

**問題点:**
- 集計関数がNULLを無視するため、COUNT(*)とCOUNT(email)が異なる値になり混乱の原因になる
- JOINでNULLは一致しないため、外部キー関係が壊れることがある
- 一部のインデックス実装ではNULLが非効率

**改善策:**
- 必須項目はNOT NULL制約を付ける
- 不明・未入力の概念は専用コード値（例: 'UNKNOWN'）や0で表現することを検討する

---

### 2.7 ゴッドテーブル（God Table）

**アンチパターン:**
```
-- 1テーブルに100列以上、「すべてを格納するマスター」
CREATE TABLE everything (col1, col2, ..., col150);
```

**問題点:**
- 列指向DBではスキャンコストは列数に比例する
- テーブルの意味（責務）が不明確になる
- スキーマ変更の影響範囲が大きすぎる

**改善策:**
- 垂直分割（縦方向の正規化）を検討する
- 異なるエンティティを同一テーブルに押し込まない

---

## 3. データマート・DWH設計のアンチパターン

### 3.1 ディメンジョンの変更履歴を記録しない（SCD未対応）

**アンチパターン:**
```sql
-- 顧客の地域が変わったら上書き更新するだけ
UPDATE dim_customers
SET region = 'East'
WHERE customer_id = 123;
```

**問題点:**
- 過去の売上データとの整合性が壊れる（顧客は実際には'West'にいたはず）
- 過去に遡った分析が不正確になる

**改善策:**
- SCD Type 2（緩やかに変化するディメンジョン）を実装する

```sql
-- SCD Type 2の例
CREATE TABLE dim_customers (
  customer_key  INT,      -- サロゲートキー
  customer_id   INT,      -- ナチュラルキー
  region        VARCHAR(50),
  valid_from    DATE,
  valid_to      DATE,
  is_current    BOOLEAN
);
```

参考: [データ・ウェアハウスの論理設計 - Oracle Docs](https://docs.oracle.com/cd/E82638_01/dwhsg/data-warehouse-logical-design.html)

---

### 3.2 スノーフレークスキーマの過剰使用

**アンチパターン:**
```
-- ディメンジョンをさらに正規化する
dim_customers → dim_cities → dim_prefectures → dim_regions
```

**問題点:**
- 分析クエリのJOIN数が増え、クエリが複雑化する
- DWHのクエリエンジンはJOINコストが高い傾向がある

**改善策:**
- BI・レポート用のデータマートには、スノーフレークよりも**スタースキーマ**（フラットなディメンジョン）を優先する

参考: [Star Schema vs Snowflake Schema: Which Is Better? - Monte Carlo Data](https://www.montecarlodata.com/blog-star-schema-vs-snowflake-schema/)

---

### 3.3 マテリアライズドビュー・中間テーブルを使わない

**アンチパターン:**
```sql
-- 毎回BIツールが重いSQLを実行する
SELECT
  date,
  SUM(revenue),
  COUNT(DISTINCT user_id)
FROM fact_orders
JOIN dim_products ON ...
JOIN dim_customers ON ...
WHERE ...
GROUP BY date;
```

**問題点:**
- 同じ重いクエリが何度も実行され、DWHのコストが増大する
- BIツールの応答が遅くなる

**改善策:**
- 集計済みマートテーブルやマテリアライズドビューを作成し、最終レイヤーはシンプルなSELECTで完結させる
- dbtなどのデータ変換ツールと組み合わせて、定期的に再構築する

---

### 3.4 データレイクのRaw層を直接BIで使う

**アンチパターン:**
```
BI Tool → S3 / GCS の生JSONをそのままクエリ
```

**問題点:**
- 型変換・NULL処理・重複排除が各BI利用者の責任になり、不整合が発生する
- パフォーマンスが不安定

**改善策:**
- Raw → Staging → Mart の3層アーキテクチャを守る
- データマートレイヤーで型・品質を保証し、BIはマートのみ参照させる

---

## 4. インデックス設計のアンチパターン

### 4.1 過剰インデックス（Over-Indexing）

**アンチパターン:**
```sql
-- 全カラムにインデックスを貼る
CREATE INDEX idx_col1 ON orders (col1);
CREATE INDEX idx_col2 ON orders (col2);
...
CREATE INDEX idx_col20 ON orders (col20);
```

**問題点:**
- INSERT/UPDATE/DELETEのたびにすべてのインデックスが更新されるため**書き込み性能が大幅低下**
- インデックス自体がディスク容量を消費する
- 実際の事例：オンラインゲームのPlayersテーブルで全列インデックスを貼ったところ、ピーク時のINSERTが著しく遅化した

**改善策:**
- クエリのWHERE句・JOIN条件・ORDER BYを分析し、本当に必要なカラムだけインデックスを作成する
- 定期的にインデックスの使用状況を確認し、不要なインデックスを削除する

参考: [Database Indexing Anti-Patterns - Medium / TDS Archive](https://medium.com/data-science/database-indexing-anti-patterns-dccb1b8ecef)

---

### 4.2 インデックス不足（Under-Indexing）

**アンチパターン:**
```sql
-- 頻繁に検索されるカラムにインデックスがない
SELECT * FROM products WHERE category_name = 'Electronics';
-- category_name にインデックスなし → フルテーブルスキャン
```

**問題点:**
- テーブルが大きくなるにつれて検索速度が線形に劣化する
- CPU使用率のスパイクが発生する

**改善策:**
- EXPLAIN/EXPLAINコマンドでクエリプランを確認し、Seq Scan（フルスキャン）が頻発しているカラムを特定してインデックスを追加する

---

### 4.3 複合インデックスの列順序の誤り

**アンチパターン:**
```sql
-- クエリが WHERE user_id = ? AND status = ? の場合
CREATE INDEX idx_status_user ON orders (status, user_id);
-- ← status の種類が少なく選択性が低い列を先頭に置いている
```

**問題点:**
- インデックスの先頭列のカーディナリティが低いと選択効率が悪い
- 左端プレフィックスルール：複合インデックスは左から順にしか使えない

**改善策:**
```sql
-- 選択性の高い列（カーディナリティが高い列）を先頭に
CREATE INDEX idx_user_status ON orders (user_id, status);
```

参考: [Index Architecture and Design Guide - Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide?view=sql-server-ver17)

---

### 4.4 OLAPでのBTreeインデックスへの依存

**アンチパターン:**
```
-- 分析用DWHにOLTP向けのBTreeインデックスを大量に貼る
```

**問題点:**
- 列指向DWHではBTreeインデックスの効果が限定的
- 代わりにパーティショニング・クラスタリング・列指向圧縮の方が効果が大きい

**改善策:**
- OLAPではColumnstore Index（SQL Server）、パーティション＋クラスタリング（BigQuery）、SORTKEY（Redshift）などDB固有の最適化機能を使う

---

## 5. パーティショニング・クラスタリングのアンチパターン

### 5.1 パーティションキーの選択ミス（データスキュー）

**アンチパターン:**
```sql
-- カーディナリティが低い列をパーティションキーに選ぶ
PARTITION BY status  -- 'active', 'inactive' の2値しかない
```

**問題点:**
- パーティションサイズが極端に偏る（データスキュー）
- 一部のスロット/ノードに処理が集中し、並列性の恩恵がなくなる

**改善策:**
- 日付・タイムスタンプ列をパーティションキーに使うのが一般的
- BigQueryでは`partition_date`や`created_at`でパーティションを切る

---

### 5.2 パーティションプルーニングが機能しないクエリ

**アンチパターン:**
```sql
-- パーティション列に関数を適用してプルーニングが効かない
WHERE DATE_TRUNC('month', created_at) = '2024-01-01'
-- ↑ パーティションが日付単位でも月次集計形式のため全スキャンになる
```

**改善策:**
```sql
-- 範囲条件で書く
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'
```

---

### 5.3 パーティションとクラスタリングの誤った使い分け

| 状況 | 推奨 |
|------|------|
| 日付フィルタが主 | パーティション（日付列） |
| 複数列でフィルタが多い | クラスタリング |
| コスト事前見積もりが必要 | パーティション（バイト数が確定） |
| 複数列の高カーディナリティ | パーティション + クラスタリング併用 |

参考:
- [Introduction to partitioned tables - BigQuery Docs](https://docs.cloud.google.com/bigquery/docs/partitioned-tables)
- [Introduction to clustered tables - BigQuery Docs](https://cloud.google.com/bigquery/docs/clustered-tables)
- [BigQuery Partitioning vs Clustering: When Each Helps - Datawise](https://datawise.dev/why-partitioning-tables-is-not-a-silver-bullet-for-bigquery-performance)

---

## 6. クラウドDWH別のアンチパターン

### 6.1 BigQuery固有のアンチパターン

#### テーブルシャーディング（日付サフィックスによるテーブル分割）

**アンチパターン:**
```
orders_20240101
orders_20240102
orders_20240103
```

**問題点:**
- BigQueryはシャーディングされたテーブルのメタデータとパーミッションを個別管理するため大きなオーバーヘッドが発生
- クエリが複雑化（`_TABLE_SUFFIX`の利用が必要）

**改善策:**
```sql
-- パーティションテーブルに移行する
CREATE TABLE orders
PARTITION BY DATE(created_at)
OPTIONS (require_partition_filter = TRUE);
```

#### ネステッド/リピーテッドフィールドの不適切な使用

**アンチパターン:**
```
-- ARRAY<STRUCT<...>> を使わずに全列をフラット化し、行数が爆発する
```

**改善策:**
- BigQueryのARRAY型/STRUCT型を活用し、1イベント1行の原則を保ちながら正規化コストを下げる

参考: [SQL Anti-Patterns for BigQuery - Towards Data Science](https://towardsdatascience.com/bigquery-anti-patterns-dacb61f8a3f/)

---

### 6.2 Amazon Redshift固有のアンチパターン

#### DISTKEYの選択ミス

**アンチパターン:**
```sql
-- カーディナリティが低い列をDISTKEYにする
DISTKEY (status)  -- 'active'/'inactive'の2値 → データスキュー確定
```

**問題点:**
- データが特定ノードに偏り、分散処理の効果がなくなる
- JOIN時にネットワーク転送（Broadcast）が発生する

**改善策:**
```sql
-- 高カーディナリティ列、かつ頻繁にJOINする列をDISTKEYに
DISTKEY (user_id)

-- 結合するファクトとディメンジョンで同じDISTKEYを使うとネットワーク転送ゼロ
```

#### SORTKEYの欠如・誤設定

**問題点:**
- SORTKEYがないと、クエリのフィルタ範囲に関係なく全ブロックをスキャンする（Zone Mapが効かない）
- 時系列データで最新データを頻繁に参照するなら、タイムスタンプ列をSORT KEY先頭列にすべき

**改善策:**
```sql
SORTKEY (event_timestamp)

-- JOINとフィルタ両方に使う列なら、DISTKEYとSORTKEYを同一列にする
DISTKEY (user_id)
SORTKEY (user_id, event_timestamp)
```

参考:
- [Best practices for designing Amazon Redshift tables - AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/best-practices-tables.html)
- [Mastering distkey and sortkey in Redshift - Airbyte](https://airbyte.com/data-engineering-resources/distkey-and-sortkey-in-redshift)

---

### 6.3 Snowflake固有のアンチパターン

#### クラスタリングキーの過剰定義

**問題点:**
- Snowflakeは自動クラスタリングを持つが、クラスタリングキーを不適切に設定すると再クラスタリングのコストが増大する

**改善策:**
- 頻繁にフィルタされる低カーディナリティの列（例: date, region）をクラスタリングキーにする
- 自動クラスタリングの有効性をAUTO_CLUSTERING_ON/OFFで制御する

#### ウェアハウスのサイズ固定（Auto-suspendを使わない）

**問題点:**
- 分析クエリのピーク/谷に関わらずウェアハウスが常時稼働してコストが増大する

**改善策:**
- Auto-suspendとAuto-resumeを設定し、アイドル時は自動停止させる

---

## 7. NULLの扱いに関するアンチパターン

### 7.1 NULL比較に`=`を使う

**アンチパターン:**
```sql
WHERE column = NULL    -- 常にFALSEを返す
WHERE column != NULL   -- NULLの行を返さない
```

**問題点:**
- NULLはどんな値とも等しくない（NULL = NULLもFALSE）
- NOT EQUALフィルタはNULL行を返さないため、除外したつもりのない行が消える

**改善策:**
```sql
WHERE column IS NULL
WHERE column IS NOT NULL
WHERE column IS DISTINCT FROM NULL  -- 一部のDB
```

---

### 7.2 集計関数のNULL無視を認識しない

**アンチパターン:**
```sql
-- ユーザー数を数えようとしているが、NULLが多い列を集計している
SELECT COUNT(email) FROM users;  -- emailがNULLの行はカウントされない
```

**改善策:**
```sql
SELECT COUNT(*) FROM users;            -- 全行数
SELECT COUNT(email) FROM users;        -- emailが非NULLの行数
SELECT COUNT(DISTINCT email) FROM users; -- 重複除去後の非NULL件数
-- → 用途に応じて使い分ける
```

---

### 7.3 NULLの四則演算

**アンチパターン:**
```sql
-- NULLが含まれると演算結果がNULLになることを忘れている
SELECT revenue + tax AS total  -- taxがNULLならtotalもNULL
```

**改善策:**
```sql
SELECT revenue + COALESCE(tax, 0) AS total
```

---

### 7.4 CASE式でNULLが漏れる

**アンチパターン:**
```sql
CASE
  WHEN status = 'active' THEN 'アクティブ'
  WHEN status = 'inactive' THEN '非アクティブ'
  -- statusがNULLの場合はELSEもないのでNULLが返る
END
```

**改善策:**
```sql
CASE
  WHEN status = 'active' THEN 'アクティブ'
  WHEN status = 'inactive' THEN '非アクティブ'
  ELSE '不明'
END
```

参考:
- [SQLアンチパターン簡単まとめ - Zenn](https://zenn.dev/yukito0616/articles/00ccc30b58e458)
- [「SQLのアンチパターン」3パターンを解説 - SINTEF](https://products.sint.co.jp/siob/blog/sql-anti-pattern)

---

## 8. チェックリスト

### クエリレビューチェックリスト

- [ ] `SELECT *` を使っていないか
- [ ] WHERE句の列に関数を適用していないか（インデックス無効化）
- [ ] 暗黙の型変換が発生していないか（リテラルの型が列型と一致しているか）
- [ ] 相関サブクエリをウィンドウ関数やJOINに置き換えられるか
- [ ] セルフジョインをウィンドウ関数に置き換えられるか
- [ ] 意図しないクロスジョインが発生していないか（JOINのON句を確認）
- [ ] HAVING句の条件をWHERE句に移動できるか
- [ ] EXPLAIN/EXPLAINプランを確認し、フルスキャンが不要に発生していないか
- [ ] LIKE検索で先頭ワイルドカードを使っていないか
- [ ] NULLの比較に`IS NULL`/`IS NOT NULL`を使っているか

### テーブル設計チェックリスト

- [ ] 1カラムに複数値を格納していないか（ジェイウォーク）
- [ ] ファクトテーブルのグレイン（粒度）が明確に定義されているか
- [ ] ディメンジョンの変更履歴（SCD）の要件を確認したか
- [ ] 過度な正規化でJOIN数が多くなっていないか（OLAP用途）
- [ ] テーブルシャーディング（日付サフィックス）ではなくパーティションを使っているか
- [ ] 必須項目にNOT NULL制約を付けているか

### インデックス・パーティションチェックリスト

- [ ] 複合インデックスの列順は選択性の高い順か
- [ ] 不要なインデックスが大量に存在しないか（書き込みコスト増）
- [ ] パーティションキー列にフィルタ条件が当たっているか（プルーニング確認）
- [ ] パーティション列に関数を適用してプルーニングを無効化していないか
- [ ] BigQuery: パーティション+クラスタリングの使い分けは適切か
- [ ] Redshift: DISTKEY/SORTKEYの設定は適切か（データスキューの確認）
- [ ] Snowflake: Auto-suspendが設定されているか

---

## 参考文献・出典

### 書籍
- [SQLアンチパターン（第2版）- O'Reilly Japan](https://www.oreilly.co.jp/books/9784814400744/)
- [SQL Antipatterns - Pragmatic Programmers](https://pragprog.com/titles/bksap1/sql-antipatterns-volume-1/)

### 日本語記事
- [データベース設計のアンチパターンとは？ - 株式会社コーソル](https://cosol.jp/column/anti-pattern/)
- [SQLアンチパターン簡単まとめ - Zenn](https://zenn.dev/yukito0616/articles/00ccc30b58e458)
- [SQLアンチパターン まとめ - Zenn](https://zenn.dev/kody/articles/31b4cc78c8434a)
- [「SQLのアンチパターン」3パターンを解説 - SINTEFブログ](https://products.sint.co.jp/siob/blog/sql-anti-pattern)
- [SQLアンチパターン「ジェイウォーク」との付き合い方 - クリーヴァ株式会社](https://note.com/creava/n/n0afb33a99728)
- [データモデリングにおける非正規化とスタースキーマ - FLINTERS BASE BLOG](https://blog.flinters-base.co.jp/entry/2022/09/28/000000)
- [今こそ注目！DWHにおけるデータモデリングとその歴史 - NTTデータ](https://www.nttdata.com/jp/ja/trends/data-insight/2022/0407/)
- [アンチパターンで学ぶDB設計 - Qiita](https://qiita.com/nem_Nuco/items/6a00e1764628b1b3fefb)

### 英語記事・ドキュメント
- [SQL Anti-Patterns for BigQuery - Towards Data Science](https://towardsdatascience.com/bigquery-anti-patterns-dacb61f8a3f/)
- [BigQuery Query Performance Optimization Guide 2025 - e6data](https://www.e6data.com/query-and-cost-optimization-hub/how-to-optimize-bigquery-query-performance)
- [Introduction to partitioned tables - BigQuery Docs](https://docs.cloud.google.com/bigquery/docs/partitioned-tables)
- [Introduction to clustered tables - BigQuery Docs](https://cloud.google.com/bigquery/docs/clustered-tables)
- [BigQuery Partitioning vs Clustering: When Each Helps - Datawise](https://datawise.dev/why-partitioning-tables-is-not-a-silver-bullet-for-bigquery-performance)
- [Best practices for designing Amazon Redshift tables - AWS](https://docs.aws.amazon.com/prescriptive-guidance/latest/query-best-practices-redshift/best-practices-tables.html)
- [Mastering distkey and sortkey in Redshift - Airbyte](https://airbyte.com/data-engineering-resources/distkey-and-sortkey-in-redshift)
- [Database Indexing Anti-Patterns - Medium / TDS Archive](https://medium.com/data-science/database-indexing-anti-patterns-dccb1b8ecef)
- [Index Architecture and Design Guide - Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/sql-server-index-design-guide?view=sql-server-ver17)
- [How (not) to kill your SQL Server performance with implicit conversion - Data Mozart](https://data-mozart.com/how-not-to-kill-your-sql-server-performance-with-implicit-conversion/)
- [SQL Anti-Patterns You Should Avoid - Data Methods Substack](https://datamethods.substack.com/p/sql-anti-patterns-you-should-avoid)
- [Star Schema Guide - MotherDuck](https://motherduck.com/learn/star-schema-data-warehouse-guide/)
- [Star Schema vs Snowflake Schema - Monte Carlo Data](https://www.montecarlodata.com/blog-star-schema-vs-snowflake-schema/)
- [Mastering SQL: 34+ Common SQL Antipatterns - Sonra.io](https://sonra.io/mastering-sql-how-to-detect-and-avoid-34-common-sql-antipatterns/)
- [Use The Index, Luke (インデックス設計の教科書)](https://use-the-index-luke.com/)
- [データ・ウェアハウスの論理設計 - Oracle Docs](https://docs.oracle.com/cd/E82638_01/dwhsg/data-warehouse-logical-design.html)
- [Finding Query Anti-Patterns - ProcureSQL](https://procuresql.com/blog/2024/01/03/query-anti-patterns-developers-sql-server-2022/)
- [SQL Server 2022: Capture SQL Anti-Patterns - Red Gate Simple Talk](https://www.red-gate.com/simple-talk/databases/sql-server/performance-sql-server/sql-server-2022-capture-sql-anti-patterns/)
