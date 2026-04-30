\# 気象庁地震データ取得・保存・分析基盤（Tableau連携 + GR則評価）提案



\## 1. ゴール



\- 気象庁の地震データを\*\*定期取得\*\*し、分析しやすい形で\*\*可読性高く保存\*\*する。

\- 特定地震（本震候補）を起点に、前後の活動を抽出して\*\*Gutenberg-Richter（GR）則\*\*を計算・可視化する。

\- Tableauで通常分析（時系列、空間分布、深さ分布など）を行い、GR則の評価も同等の視認性で確認できるようにする。

\- ローカルCodexで引き継ぎ可能な、運用しやすい構成にする。



\---



\## 2. 全体アーキテクチャ（最小構成）



```text

\[JMA feed/API/CSV]

&#x20;     ↓ (定期バッチ)

\[ingest: 取得・検証・差分判定]

&#x20;     ↓

\[raw層: 生データ保存(JSON/CSV)]

&#x20;     ↓

\[normalize層: スキーマ統一・品質補正]

&#x20;     ↓

\[curated層: Tableau用ファクトテーブル]

&#x20;     ↓

\[Tableauダッシュボード]

&#x20;     ↘

&#x20;     \[GR則計算モジュール: Python]

&#x20;           ↓

&#x20;     \[GR結果テーブル + 図表出力]

```



\### 推奨技術（ローカル運用前提）



\- 実装: Python 3.11+

\- 取得/整形: `requests`, `pandas`, `pydantic`

\- 保存: まずは `Parquet + SQLite`（軽量）

\- スケジューリング: cron（Linux/macOS）またはタスクスケジューラ（Windows）

\- 可視化: Tableau（抽出更新）



> 将来、データ量増大時は PostgreSQL + Airflow / Prefect へ移行しやすいように、取得・整形・計算をモジュール分割する。



\---



\## 3. データモデル（実運用で効く設計）



\### 3.1 raw層（再現性重視）



\- `raw\_earthquake\_events`

&#x20; - `source\_name`（JMAなど）

&#x20; - `fetched\_at\_utc`

&#x20; - `source\_event\_id`

&#x20; - `payload\_json`（原文）

&#x20; - `payload\_hash`（重複排除）



\### 3.2 normalized層（分析前の標準化）



\- `eq\_events\_normalized`

&#x20; - `event\_id`（内部一意キー）

&#x20; - `origin\_time\_jst`

&#x20; - `latitude`, `longitude`, `depth\_km`

&#x20; - `magnitude`, `magnitude\_type`

&#x20; - `region\_name`

&#x20; - `max\_intensity`（震度）

&#x20; - `status`（速報/確定など）

&#x20; - `revision\_no`（同一イベント改訂追跡）



\### 3.3 curated層（Tableau最適化）



\- `fact\_earthquake`

&#x20; - Tableauで使う主要列を1テーブル化

\- `dim\_region`, `dim\_time`, `dim\_intensity`（任意）

\- `fact\_gr\_window`

&#x20; - 特定地震ごとのGR解析結果（後述）



\---



\## 4. 定期取得の設計



\## 4.1 取得頻度



\- 通常監視: 5〜10分間隔

\- コストを抑えるなら 30分〜1時間間隔でも開始可能



\## 4.2 差分取得



\- `source\_event\_id + revision\_no` をキーにUPSERT

\- 改訂が入るデータを考慮し、\*\*同一イベントの更新履歴を保持\*\*



\## 4.3 品質チェック



\- 必須列（時刻、緯度経度、M）の欠損率

\- 時刻の逆転（更新時の異常）

\- 異常値（M < -1, depth < 0 など）の隔離



\---



\## 5. GR則評価の実装方針



GR則（log10 N = a - bM）は、解析窓の切り方と完全性マグニチュードMcの推定が重要です。



\### 5.1 分析単位



\- 基準地震（target event）を選択

\- 時間窓（例: 発生前30日 / 後30日）

\- 空間窓（例: 震央から半径50km）

\- 深さ制約（例: 0〜100km）



\### 5.2 計算ステップ



1\. 窓内イベント抽出

2\. Mc推定（MAXC法など）

3\. M >= Mc でGR回帰

4\. `a値`, `b値`, `R^2`, サンプル数を保存

5\. bootstrapでb値の信頼区間算出（推奨）



\### 5.3 前震・余震の「参考判定」



厳密分類は難しいため、運用上は以下のスコア化を推奨。



\- `pre\_activity\_score`

&#x20; - 本震前窓でのb値低下、イベント密度上昇

\- `post\_activity\_score`

&#x20; - 本震後窓でのOmori的減衰傾向 + GR適合度



> 判定は「参考指標」であり、確定ラベルではない扱いにする（UIにも注記）。



\---



\## 6. Tableau可視化案



\### ダッシュボードA: 地震モニタリング



\- 地図: 震央プロット（サイズ=M、色=深さor震度）

\- 時系列: 日次件数 + 累積件数

\- フィルタ: 期間、地域、深さ、M



\### ダッシュボードB: GR則評価



\- Magnitude-Frequency プロット（log N vs M）

\- 回帰線（M>=Mc）

\- 指標カード: Mc, a, b, 95%CI, N

\- 前/後比較: 本震前窓 vs 本震後窓を並列表示



\### ダッシュボードC: 個別地震ビュー



\- 対象イベントの詳細

\- 前後窓の空間分布ヒートマップ

\- 「参考判定スコア」ゲージ



\---



\## 7. 実装ロードマップ（Codex引き継ぎ向け）



\### Phase 1（1〜2週間）: 基盤作成



\- 取得スクリプト（定期実行）

\- raw/normalizedテーブル作成

\- 重複排除・改訂追跡

\- Tableauで最低限の地図/時系列表示



\### Phase 2（1〜2週間）: GR則モジュール



\- イベント窓抽出

\- Mc推定 + b値計算

\- 結果テーブル化（`fact\_gr\_window`）

\- TableauでGRダッシュボード



\### Phase 3（継続）: 精度改善



\- しきい値チューニング

\- ブートストラップ導入

\- アラート（閾値超過で通知）



\---



\## 8. ディレクトリ構成例（ローカルCodex運用）



```text

project/

&#x20; data/

&#x20;   raw/

&#x20;   normalized/

&#x20;   curated/

&#x20; src/

&#x20;   ingest/

&#x20;     fetch\_jma.py

&#x20;     normalize.py

&#x20;   analytics/

&#x20;     gr\_law.py

&#x20;     windowing.py

&#x20;   pipelines/

&#x20;     run\_ingest.py

&#x20;     run\_gr.py

&#x20; tableau/

&#x20;   datasource\_notes.md

&#x20; docs/

&#x20;   data\_dictionary.md

&#x20;   operation\_runbook.md

```



\---



\## 9. 運用上の注意



\- 地震データは改訂されるため、\*\*追記専用ログ + 最新ビュー\*\*の二層管理が安全。

\- 地域・期間によってMcが変動し得るため、固定値運用は避ける。

\- 解析結果は防災判断の確定根拠ではなく、補助情報として扱う。



\---



\## 10. すぐ着手できる最初の一歩



1\. JMAデータ取得元を確定（フォーマット確認）

2\. `raw\_earthquake\_events` と `eq\_events\_normalized` のDDLを作成

3\. 5分間隔の取得ジョブを立てる

4\. Tableauで `fact\_earthquake` を接続

5\. target eventを1件選び、GR試算を手計算で検証



これで「取れる・見える・試せる」状態を最短で作れます。

