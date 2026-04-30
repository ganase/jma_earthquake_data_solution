# 気象庁地震データ取得・保存・分析基盤提案

## 1. ゴール

本提案のゴールは、気象庁が公開する地震データを継続的に取得し、後続分析で扱いやすい形に保存し、データ分析ソフトウェアによる監視・探索分析と、特定地震の前後における Gutenberg-Richter 則（以下、GR則）評価を行える基盤を整備することである。

- 気象庁地震データを 1日1回の間隔で定期取得する。
- raw データを改変せず保存し、分析向けに normalized / curated 層へ整形する。
- データ分析ソフトウェアから接続しやすいファクトテーブルを提供する。
- 特定地震の前後で GR則を評価し、前震・余震活動を考えるための参考指標を提供する。
- ローカル Codex が引き継ぎやすい、責務が分離されたディレクトリ構成にする。

## 2. 全体アーキテクチャ

データ処理は、取得、保存、正規化、分析用整形、可視化を段階的に分ける。

```text
JMA source
  -> ingest
  -> raw
  -> normalized
  -> curated
  -> データ分析ソフトウェア
```

GR則の計算は、通常のデータ取得・整形パイプラインから分離したモジュールとして実装する。

```text
curated earthquake events
  -> GR calculation module
  -> fact_gr_window
  -> データ分析ソフトウェア GR評価ダッシュボード
```

責務の分離方針は以下とする。

- `ingest`: 気象庁データの取得、取得時刻記録、差分判定。
- `raw`: 取得した原文または原データを再現性重視で保存。
- `normalized`: 時刻、座標、深さ、マグニチュード、地域名などを標準スキーマへ変換。
- `curated`: データ分析ソフトウェアや GR 計算で利用しやすい形へ集約。
- `gr`: windowing、Mc 推定、b値推定、信頼区間計算を担当。

## 3. データモデル

### 3.1 `raw_earthquake_events`

取得したデータを可能な限り原形に近い状態で保存するテーブル。

| Column | Description |
| --- | --- |
| `raw_id` | raw レコードの内部 ID |
| `source_name` | データ取得元。例: `jma` |
| `source_event_id` | 取得元が持つイベント ID。存在しない場合は時刻・位置などから生成 |
| `fetched_at_utc` | 取得実行時刻 |
| `payload_json` | 原文または取得結果を JSON 化したもの |
| `payload_hash` | 重複検出用ハッシュ |
| `revision_key` | 改訂追跡用キー |

### 3.2 `eq_events_normalized`

地震イベントを分析可能な標準形式へ変換したテーブル。

| Column | Description |
| --- | --- |
| `event_id` | 内部イベント ID |
| `source_event_id` | 取得元イベント ID |
| `origin_time_jst` | 発震時刻（JST） |
| `origin_time_utc` | 発震時刻（UTC） |
| `latitude` | 緯度 |
| `longitude` | 経度 |
| `depth_km` | 深さ km |
| `magnitude` | マグニチュード |
| `magnitude_type` | Mj などの種別 |
| `region_name` | 震央地名・地域名 |
| `max_intensity` | 最大震度 |
| `status` | 速報、暫定、確定など |
| `revision_no` | 同一イベント内の改訂番号 |
| `is_latest_revision` | 最新改訂かどうか |

### 3.3 `fact_earthquake`

データ分析ソフトウェアから直接扱う主ファクトテーブル。

| Column | Description |
| --- | --- |
| `event_id` | 内部イベント ID |
| `origin_date_jst` | JST 日付 |
| `origin_time_jst` | JST 発震時刻 |
| `latitude` | 緯度 |
| `longitude` | 経度 |
| `depth_km` | 深さ km |
| `magnitude` | マグニチュード |
| `region_name` | 地域名 |
| `max_intensity` | 最大震度 |
| `revision_no` | 改訂番号 |

必要に応じて `dim_time`、`dim_region`、`dim_intensity` を追加する。

### 3.4 `fact_gr_window`

GR則評価の計算結果を保存するテーブル。

| Column | Description |
| --- | --- |
| `gr_window_id` | GR 計算窓 ID |
| `target_event_id` | 基準とする地震イベント ID |
| `window_type` | `before` / `after` / `custom` |
| `window_start_utc` | 計算窓開始 |
| `window_end_utc` | 計算窓終了 |
| `radius_km` | 空間窓の半径 |
| `depth_min_km` | 深さ下限 |
| `depth_max_km` | 深さ上限 |
| `event_count` | 窓内イベント数 |
| `mc` | 完全性マグニチュード |
| `a_value` | GR則の a値 |
| `b_value` | GR則の b値 |
| `b_ci_low` | b値信頼区間の下限 |
| `b_ci_high` | b値信頼区間の上限 |
| `fit_r2` | 回帰の適合度 |
| `method` | Mc 推定・b値推定の手法 |

## 4. 定期取得設計

### 4.1 取得頻度

初期運用では 1日1回の取得を基本案とする。速報性よりも安定運用と日次分析の継続性を優先する。

取得ジョブは以下を満たすようにする。

- 前回取得時刻以降のデータを差分取得する。
- 取得元が差分 API を持たない場合は、一定期間を再取得し、UPSERT で重複を吸収する。
- 取得実行ごとに `fetched_at_utc` と取得件数を記録する。

### 4.2 UPSERT と revision 追跡

気象庁データは後から改訂される可能性があるため、単純な上書きだけにしない。

- `source_event_id` と `revision_key` を使って UPSERT する。
- 原文の `payload_hash` が変わった場合は、新しい revision として保存する。
- `eq_events_normalized` では `revision_no` と `is_latest_revision` を持つ。
- データ分析ソフトウェア用の `fact_earthquake` は、原則として最新 revision のみを参照する。
- 監査や再計算のために、raw 層では過去 revision を保持する。

### 4.3 欠損・異常値の品質チェック

取得後、最低限以下の品質チェックを実施する。

- 発震時刻、緯度、経度、マグニチュードの欠損率。
- 緯度・経度が日本周辺として明らかに不自然でないか。
- `depth_km < 0`、極端に大きい深さなどの異常値。
- `magnitude` が欠損または分析対象外の形式でないか。
- 同一イベントが過剰に重複していないか。
- 取得件数が直近平均から大きく外れていないか。

品質チェック結果は運用ログに残し、データ分析ソフトウェアの監視ダッシュボードにも表示できる形にする。

## 5. GR則評価設計

GR則は以下の形で地震の規模別頻度を表す。

```text
log10 N = a - bM
```

ここでの評価は、前震・余震を確定分類するものではなく、活動変化を確認するための参考指標として扱う。

### 5.1 windowing

基準地震 `target_event_id` を指定し、時間・空間・深さの窓で対象イベントを抽出する。

- 時間窓: 例として発生前 30 日、発生後 30 日、発生後 7 日など。
- 空間窓: 震央から半径 30 km、50 km、100 km など。
- 深さ窓: 0〜100 km、または基準地震の深さ ±20 km など。
- マグニチュード範囲: Mc 推定後、`M >= Mc` を GR 回帰対象にする。

同じ基準地震に対して複数の窓を保存し、データ分析ソフトウェアで比較できるようにする。

### 5.2 Mc 推定

Mc（完全性マグニチュード）は固定値にせず、地域・期間・データ密度ごとに推定する。

初期実装では、実装しやすい MAXC 法を候補とする。データ量が増えた段階で、Goodness-of-Fit 法や EMR 法との比較を検討する。

保存すべき情報は以下。

- 推定された `mc`
- 推定手法
- 推定に使用したイベント数
- Mc 未満を除外した後のイベント数

### 5.3 b値推定

初期実装では、Mc 以上のイベントを対象に最尤推定または線形回帰で b値を求める。継続運用では、マグニチュードの丸め幅を考慮した推定式を採用する。

計算結果として以下を保存する。

- `a_value`
- `b_value`
- `fit_r2`
- `event_count`
- `method`

### 5.4 信頼区間

b値はサンプルサイズや窓の切り方に影響されるため、信頼区間を必ず併記する。初期実装では bootstrap による 95% 信頼区間を推奨する。

- 窓内イベントを復元抽出する。
- 各サンプルで Mc と b値を再計算する。
- b値分布の 2.5% 点と 97.5% 点を保存する。

### 5.5 注意書き

GR則評価は、防災判断や地震予知の確定根拠ではない。データ分析ソフトウェア上でも「参考指標」「補助情報」であることを明示し、前震・余震というラベルを断定的に表示しない。

## 6. データ分析ソフトウェア可視化案

### 6.1 監視ダッシュボード

- 直近取得時刻、取得件数、エラー件数。
- 地図上の震央プロット。サイズはマグニチュード、色は深さまたは震度。
- 日次・時間別の地震件数。
- マグニチュード分布、深さ分布。
- 地域、期間、深さ、マグニチュード、最大震度のフィルタ。

### 6.2 GR評価ダッシュボード

- Magnitude-Frequency プロット。
- Mc の位置を示す参照線。
- `M >= Mc` に対する GR 回帰線。
- b値、a値、Mc、イベント数、95% 信頼区間のカード。
- 基準地震の前後窓を横並びで比較。
- 窓条件（時間、半径、深さ）をパラメータで切り替え。

### 6.3 個別地震ビュー

- 基準地震の詳細情報。
- 基準地震前後のイベント一覧。
- 前後窓の地図、時系列、深さ分布。
- GR則評価結果と bootstrap 信頼区間。
- 「前震・余震の確定判定ではなく参考指標」という注意表示。

## 7. 実装ロードマップ

### Phase 1: 取得・保存・データ分析ソフトウェア基盤

- 気象庁データ取得元と取得形式を確定する。
- `raw_earthquake_events` を作成する。
- `eq_events_normalized` を作成する。
- 1日1回の取得ジョブを用意する。
- UPSERT と revision 追跡を実装する。
- `fact_earthquake` を作成し、データ分析ソフトウェアで地図・時系列を確認する。

### Phase 2: GR則評価モジュール

- target event を指定するインターフェースを用意する。
- 時間・空間・深さ windowing を実装する。
- Mc 推定を実装する。
- b値推定と bootstrap 信頼区間を実装する。
- `fact_gr_window` を作成し、データ分析ソフトウェアで GR評価ダッシュボードを構築する。

### Phase 3: 運用強化・精度改善

- 品質チェック結果を監視ダッシュボードへ連携する。
- Mc 推定手法を複数比較できるようにする。
- 窓条件の感度分析を追加する。
- データ改訂時の GR 再計算ジョブを実装する。
- 運用 runbook と障害時の再実行手順を整備する。

## 8. ローカル Codex 引き継ぎ用ディレクトリ構成例

```text
project/
  data/
    raw/
    normalized/
    curated/
  docs/
    jma_earthquake_data_solution.md
    data_dictionary.md
    operation_runbook.md
  src/
    ingest/
      fetch_jma.py
      parse_jma.py
      upsert_raw.py
    normalize/
      normalize_events.py
      quality_checks.py
    analytics/
      gr_law.py
      mc_estimation.py
      windowing.py
      bootstrap.py
    pipelines/
      run_ingest.py
      run_normalize.py
      run_gr.py
  sql/
    raw_earthquake_events.sql
    eq_events_normalized.sql
    fact_earthquake.sql
    fact_gr_window.sql
  data_analysis_software/
    datasource_notes.md
    dashboard_specs.md
  tests/
    test_normalize_events.py
    test_gr_law.py
    test_windowing.py
```

この構成では、取得、正規化、GR計算、データ分析ソフトウェア連携を別ディレクトリに分ける。ローカル Codex が後続作業を行う際も、修正対象を絞りやすい。

## 9. 運用上の注意

- 気象庁データは改訂される可能性があるため、raw 層では過去 revision を保持する。
- データ分析ソフトウェアで見る最新データと、監査用の履歴データを分けて扱う。
- Mc は地域、期間、観測条件で変わるため、固定 Mc の運用は避ける。
- b値や GR 適合度は、イベント数が少ない窓では不安定になりやすい。
- GR則評価は、防災判断の確定根拠ではなく補助情報として扱う。
- 取得失敗や異常件数を検知できるよう、パイプライン自体の監視を用意する。
- データ改訂が入った場合は、関連する `fact_earthquake` と `fact_gr_window` を再生成できるようにする。

## 10. 最初に着手する実装候補

1. 気象庁データの取得元とフォーマットを確定する。
2. `raw_earthquake_events` と `eq_events_normalized` の DDL を作成する。
3. 1日1回で動く取得ジョブを作る。
4. `fact_earthquake` を作成し、データ分析ソフトウェアで地図と時系列を確認する。
5. 代表的な target event を 1 件選び、GR則評価を手計算に近い形で検証する。
