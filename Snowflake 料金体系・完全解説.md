# Snowflake 料金体系・完全解説

（2025年7月版・実務運用者向け）

---

## 目次

- [Snowflake 料金体系・完全解説](#snowflake-料金体系完全解説)
  - [目次](#目次)
  - [1. Snowflake課金の全体像](#1-snowflake課金の全体像)
  - [2. 各課金対象の正体とロジック](#2-各課金対象の正体とロジック)
    - [2.1 Storage（ストレージ）](#21-storageストレージ)
    - [2.2 Compute（計算リソース）](#22-compute計算リソース)
      - [2.2.1 仮想ウェアハウス（Virtual Warehouse, VW）](#221-仮想ウェアハウスvirtual-warehouse-vw)
      - [2.2.2 サーバーレス Compute（Serverless）](#222-サーバーレス-computeserverless)
    - [2.3 Cloud Services（クラウドサービス層）](#23-cloud-servicesクラウドサービス層)
  - [3. 実際のクレジット消費の流れ](#3-実際のクレジット消費の流れ)
  - [4. 課金の計算例・パターン別比較](#4-課金の計算例パターン別比較)
    - [4.1 仮想ウェアハウス：連続実行と間欠実行](#41-仮想ウェアハウス連続実行と間欠実行)
      - [パターンA：1分以内に1秒クエリ×60本をまとめて実行](#パターンa1分以内に1秒クエリ60本をまとめて実行)
      - [パターンB：1分おきに1本ずつ（Auto-Suspend60秒）](#パターンb1分おきに1本ずつauto-suspend60秒)
    - [4.2 サーバーレス](#42-サーバーレス)
    - [4.3 Cloud Servicesの課金例](#43-cloud-servicesの課金例)
  - [5. コストを最適化する実践Tips](#5-コストを最適化する実践tips)
  - [6. 運用現場で必ず役立つ確認・分析方法](#6-運用現場で必ず役立つ確認分析方法)
    - [6.1 Snowsightで可視化](#61-snowsightで可視化)
    - [6.2 SQLで実績集計](#62-sqlで実績集計)
  - [7. よくある誤解・落とし穴](#7-よくある誤解落とし穴)
  - [8. まとめ](#8-まとめ)

---

## 1. Snowflake課金の全体像

Snowflakeの利用料金は「クレジット（Credit）」という仮想通貨で測り、\*\*「どれだけ計算したか」「どれだけ保存したか」「どれだけクラウド内部サービスを使ったか」\*\*によって最終的な課金額が決まります。

主な課金カテゴリは3つです。

- **Storage**：保存データ量に比例（TB-月単位）
- **Compute**：クエリ/ETL等の計算リソース利用量（秒単位）
- **Cloud Services**：クエリ解析や認証などのメタデータ処理（Computeの10％超え分のみ課金）

---

## 2. 各課金対象の正体とロジック

### 2.1 Storage（ストレージ）

- **何に課金？**
  　圧縮後のデータ保存量（TB-月）。未圧縮ではなく**圧縮後**サイズ。
- **どのくらい？**
  　1TB-月あたり約\$23（東京リージョン例、契約・為替で変動）。数百GBなら月数ドル程度。
- **注意点**
  　テーブル履歴、Time TravelやFail-safe用データも課金対象。

---

### 2.2 Compute（計算リソース）

Snowflakeの「計算資源」は2つの形態があります。

#### 2.2.1 仮想ウェアハウス（Virtual Warehouse, VW）

- **特徴**
  　ユーザ自身で起動/停止・サイズ（X-Small～6X-Large）・Auto-Suspend秒数を設定。

- **課金単位**
  　**秒単位課金、ただし再起動ごとに最小60秒ぶん一括課金（60秒ルール）**
  　例：VW再開→10秒で処理終了→60秒分請求。61秒動作→61秒分課金。

- **サイズごとの料金例**（Standardプラン）

  | サイズ     | 1時間あたり | 1秒あたり      | 60秒分     |
  | ------- | ------ | ---------- | -------- |
  | X-Small | 1cr    | 0.000278cr | 0.0167cr |
  | Small   | 2cr    | 0.000556cr | 0.0333cr |
  | Medium  | 4cr    | 0.001111cr | 0.0667cr |

- **運用の注意**
  　VW再開が多いほど60秒課金が積み上がる。Auto-Suspendを短くしすぎると高コスト化。

#### 2.2.2 サーバーレス Compute（Serverless）

- **特徴**
  　Snowflake側で自動的に計算資源を立ち上げ。ユーザーは一切サーバ管理不要。
- **主な用途**
  　**Snowpipe**（データ自動ロード）、**Tasks**（自動バッチ処理）、**Search Optimization Service** など。
- **課金単位**
  　**実際に使った秒数分のみ課金（最小1秒、60秒ルール無し）**
- **メリット**
  　「細切れ＆短時間処理」や「ファイル着信イベント」などに極めて効率的。

---

### 2.3 Cloud Services（クラウドサービス層）

- **何に課金？**
  　認証、クエリ解析・実行計画、カタログやメタデータ管理など“裏方”の処理。
- **課金ロジック**
  　**「当日のCompute（VW/Serverless両方）の合計消費クレジットの10％まで」無料**。
  　超えた分だけ課金。

  - 例：Compute合計100cr、CloudServices15crなら、(15-10)=5crだけ請求。
  - 逆にCompute0でCloudServicesだけ発生した場合は全額請求される点に注意。

---

## 3. 実際のクレジット消費の流れ

- クエリ/ロード等の実行
  　→ Compute (VW or Serverless) が消費される
- 付随する内部処理（認証・最適化等）がCloudServicesで消費される
- 日次で集計し、**Cloud Services10%超過分**だけ追加で請求
- ストレージ・データ転送は月次で別請求

---

## 4. 課金の計算例・パターン別比較

### 4.1 仮想ウェアハウス：連続実行と間欠実行

#### パターンA：1分以内に1秒クエリ×60本をまとめて実行

- VW再開1回→60秒で全て終了
- 請求：**60秒分のみ**

#### パターンB：1分おきに1本ずつ（Auto-Suspend60秒）

- 毎回VW再開→各60秒請求→60回×60秒＝3,600秒（60分）分課金
- 請求：**60分分**

→ まとめて処理した方が圧倒的に安い！

### 4.2 サーバーレス

- 60本を1秒ずつ処理しても**実処理時間（合計60秒）分だけ課金**、アイドル時間・再開オーバーヘッドなし。

### 4.3 Cloud Servicesの課金例

- Compute 100cr、CloudServices 8cr →追加請求0（10%未満）
- Compute 100cr、CloudServices 20cr→追加請求10cr（20-10）

---

## 5. コストを最適化する実践Tips

1. **Auto-Suspend設定の見直し**
   　- 頻繁な停止/再開を避け、数分〜10分単位が目安
2. **短時間処理はServerlessへ移行**
   　- Snowpipe/Tasksを使うと60秒ルールの無駄を防げる
3. **バッチ化＆ファイルまとめ処理**
   　- Snowpipeは1ファイルごとに0.06cr/1,000filesの手数料もあるため、極端な小分けは損
4. **モニタリングを活用**
   　- `WAREHOUSE_METERING_HISTORY`でVW再開回数と秒数を把握し、無駄を削減
5. **Cloud Services消費の抑制**
   　- SHOW/DDL/INFORMATION\_SCHEMAなど「内部的に多くのメタデータを触るクエリ」はなるべくまとめる・頻度を下げる

---

## 6. 運用現場で必ず役立つ確認・分析方法

### 6.1 Snowsightで可視化

- 「Admin」→「Cost Management」→「Usage」

  - Compute・Serverless・CloudServicesの消費がカテゴリ別に確認可能

### 6.2 SQLで実績集計

- 仮想ウェアハウス消費詳細

```sql
SELECT
  WAREHOUSE_NAME,
  START_TIME,
  END_TIME,
  CREDITS_USED_COMPUTE,
  CREDITS_USED_CLOUD_SERVICES
FROM
  SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE
  START_TIME >= DATEADD(day, -7, CURRENT_TIMESTAMP())
ORDER BY
  START_TIME DESC;
```

- 日次合計クレジット

```sql
SELECT
  USAGE_DATE,
  SUM(CREDITS_USED) AS total_credits
FROM
  SNOWFLAKE.ACCOUNT_USAGE.METERING_DAILY_HISTORY
WHERE
  USAGE_DATE >= DATEADD(day, -30, CURRENT_DATE())
GROUP BY
  USAGE_DATE
ORDER BY
  USAGE_DATE DESC;
```

- サーバーレス消費は`METERING_DAILY_HISTORY`のRESOURCE\_TYPE列が`SERVERLESS_*`で抽出

---

## 7. よくある誤解・落とし穴

- **CPU使用率や処理効率は課金額に直接影響しない**
  　→ 実際に動いていた秒数と倉庫サイズだけで決まる
- **Cloud Services10%ルールは“CPU使用率が10%超えたら課金”ではない**
  　→ Computeクレジット比での「値引き枠」でしかない
- **VWを短時間頻繁に再開する運用は高コストの元**
  　→ サスペンドを長めに／バッチ化／Serverless活用
- **Serverlessだけ運用だとCloud Services全額課金に注意**
  　→ “10%の値引き枠”はComputeの利用があって初めて効く
- **ストレージ課金は圧縮後サイズ。TimeTravelやFail-safeの履歴にも課金**
  　→ 削除やパージ運用も重要

---

## 8. まとめ

- Snowflakeの料金は**ストレージ、コンピュート（VW or Serverless）、Cloud Services**で決まる
- 仮想ウェアハウスは**再開ごとに60秒最低課金**あり、再開回数がコストに直結
- サーバーレスは**1秒単位課金**で無駄が出にくい
- **Cloud Servicesは“Computeクレジットの10％超え分だけ”課金**
- コストを抑えるには**まとめて処理・Auto-Suspend調整・Serverless活用**が重要
- **運用状況はSnowsight/SQLで即時チェック可能**。実績を可視化して最適な運用設計へ
