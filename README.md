# jma_earthquake_data_solution

気象庁地震データの取得・保存・分析基盤を検討するためのリポジトリです。

現在は、データ分析ソフトウェア連携と Gutenberg-Richter 則（GR則）評価を含む提案ドキュメントを管理しています。

## Documents

- [気象庁地震データ取得・保存・分析基盤提案](docs/jma_earthquake_data_solution.md)

## Local Setup

### 1. Clone

```powershell
git clone https://github.com/ganase/jma_earthquake_data_solution.git
cd jma_earthquake_data_solution
```

### 2. Confirm Repository

作業前に、正しいリポジトリを参照していることを確認します。

```powershell
pwd
git remote -v
git rev-parse --show-toplevel
```

期待する remote は以下です。

```text
origin  https://github.com/ganase/jma_earthquake_data_solution.git
```

### 3. Check Working Tree

```powershell
git status --short --branch
```

## How to Use

### Read the Proposal

このリポジトリは現時点ではドキュメント中心です。まず以下を確認してください。

```powershell
Get-Content docs/jma_earthquake_data_solution.md
```

macOS / Linux の場合:

```bash
cat docs/jma_earthquake_data_solution.md
```

Markdown preview が使えるエディタでは、`docs/jma_earthquake_data_solution.md` を開いて確認してください。

### Current Execution Status

現時点では、地震データ取得・正規化・GR則計算を実行する Python コードはまだ含まれていません。そのため、ローカルで実行するアプリケーションやバッチ処理はありません。

提案ドキュメントでは、今後以下のような構成で実装することを想定しています。

```text
src/
  ingest/
  normalize/
  analytics/
  pipelines/
```

将来の実装後は、以下のようなコマンドで実行できる構成にする想定です。

```powershell
python -m src.pipelines.run_ingest
python -m src.pipelines.run_normalize
python -m src.pipelines.run_gr
```

## Development Notes

- `main` ブランチを最新化してから作業してください。
- 提案内容を更新する場合は、`docs/jma_earthquake_data_solution.md` を編集してください。
- 実装コードを追加する場合は、README のセットアップ手順と実行コマンドも合わせて更新してください。

```powershell
git fetch origin main
git switch main
git merge --ff-only origin/main
```
