# 記事レビュー チェックリスト

対象: `articles/20260217_github-copilot-sdk-pr-summary-bot.md`

---

## 🔴 必須（公開ブロッカー）

- [x] タイトルを `# Sreakeブログ` から適切な記事タイトルに変更
- [ ] 第6章にスクリーンショットまたは実行出力例を追加（実行が必要）
- [x] Copilot SDK の認証フロー（ローカル `gh` CLI 認証）を説明・GitHub Actions が使えない理由も記述
- [x] コードスニペットに `build_prompt` と `main()` を追加

## 🟠 高優先度

- [x] 実ワークフローファイル (`pr-summary-bot/.github/workflows/pr-summary.yml`) を削除（Actions非対応のため）
- [x] `uv venv --python 3.12` の不要ステップを削除
- [x] `pyproject.toml` の dev 依存を `[dependency-groups]` に移して `uv sync --no-dev` を正しく動かす
- [x] `system_message` の `"mode": "replace"` にリスク注記を追加
- [x] `available_tools` の説明を正確に修正（組み込みツールの許可リストである旨）

## 🟡 中優先度

- [x] 前提条件（1.2節）から `Node.js 20以上` を削除し `Python 3.12以上 / uv` に修正
- [x] 設計（3章）の「レビュー観点」→「レビューポイント」に統一
- [x] コードスニペット末尾の `asyncio.run(main())` を整理
- [x] 参考リンクにリンクテキストを追加

## ✅ 完了済み

- Section 3 にマーメイド シーケンス図を追加
- Section 5 を GitHub Actions → ローカル実行の説明に書き替え（Actions が使えない理由も記述）
- `pr-summary-bot/.github/workflows/pr-summary.yml` を削除
- タイトルを `# Sreakeブログ` → `# GitHub Copilot SDKでPR要約botを作る` に変更
- 前提条件（1.2節）から `Node.js 20以上` を削除し `Python 3.12以上 / uv` に修正
- `uv venv --python 3.12` の不要ステップを削除
- 設計（3章）の「レビュー観点」→「レビューポイント」に統一
- `build_prompt` と `main()` をコードスニペットに追加（4.2節を新設）
- `asyncio.run(main())` を `if __name__ == "__main__": asyncio.run(main())` に整理
- `system_message` の `"mode": "replace"` にリスク注記を追加
- `available_tools` の説明を正確に修正
- 参考リンクにリンクテキストを追加
- `pyproject.toml` の dev 依存を `[dependency-groups]` に移して `uv sync --no-dev` が正しく動くようにした
- 実ワークフローファイル (`pr-summary-bot/.github/workflows/pr-summary.yml`) を記事と整合させた（`npm install -g @anthropic-ai/claude-code` の誤った行を削除、uv使用に統一）
