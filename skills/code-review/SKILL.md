---
name: code-review
description: コードレビューを実施するスキル。コードの品質、可読性、保守性、セキュリティを評価し、改善提案を行う。「コードをレビューして」「このPRを確認して」「コードの問題点を指摘して」などのリクエストで使用。
context: fork
---

# Code Review Skill

## レビュープロセス

1. **全体把握** — コードの目的と構造を理解
2. **観点別チェック** — 下記チェックリストに沿って評価
3. **フィードバック作成** — 重要度付きで改善提案

## チェックリスト

### 🔴 MUST（必須で指摘）

**MUST を出す前に「指摘している『悪い状態』が実際に到達可能か」を verify する**。false positive な MUST は reviewer の信頼を消耗するため、SHOULD / MAY より厳しい precondition を課す。以下 3 点をすべて確認してから MUST 認定する:

1. **意図された挙動ではないか** — 対応する test / spec / docs が「その挙動を許可している」と明示していないか確認する。許可されていれば bug ではなく仕様であり、MUST ではない。
2. **upstream で既にブロックされていないか** — 「ガード / バリデーション / 保護が無い」と書く前に、その悪い状態への到達経路を trace する。型 / schema / enum / middleware / 別 helper / DB 制約のいずれかが先に reject していれば、ガード追加は不要 (= MUST にしない)。負の不変条件 (「X は起きない」) は test で verify されないことが多いので、helper / schema の側を読む。
3. **PR description / 要約を鵜呑みにしない** — 「実装した」と書かれていても実装の証明にはならない。必ず該当コードを直接読んで verify する。

1〜3 のいずれかをクリアできない指摘は、SHOULD に下げるか、「〜の意図を確認したい」という観察コメントに留める。

| 観点 | 確認内容 |
|------|----------|
| 正確性 | ロジックにバグはないか、エッジケースは考慮されているか |
| セキュリティ | 入力検証、認証・認可、機密情報の露出はないか |
| エラー処理 | 例外が適切にハンドリングされているか |

### 🟡 SHOULD（推奨で指摘）

| 観点 | 確認内容 |
|------|----------|
| 可読性 | 命名は意図を表しているか、複雑すぎないか |
| 重複 | DRY原則に違反していないか |
| テスタビリティ | 単体テストを書きやすい構造か |

### 🟢 MAY（任意・提案レベル）

| 観点 | 確認内容 |
|------|----------|
| パフォーマンス | 明らかな非効率はないか（早すぎる最適化は避ける） |
| スタイル | 一貫性のあるフォーマットか |

## 出力フォーマット

```markdown
## レビューサマリー
[全体的な評価を1-2文で]

## 指摘事項

### 🔴 [MUST] タイトル
- **場所**: ファイル名:行番号
- **問題**: 何が問題か
- **提案**: どう修正すべきか
- **理由**: なぜ問題か

### 🟡 [SHOULD] タイトル
...

## 良い点
[ポジティブなフィードバックも含める]
```

## GitHub PR連携

PR番号またはPR URLが引数として渡された場合、レビュー結果をGitHub PRにコメントとして投稿する。
GitHub APIとの連携はすべて `gh` CLI に統一する（MCP GitHub ツールは使用しない）。

### 手順

1. PR情報と差分を取得:

```bash
# PR情報の取得
gh pr view {pr_number} --repo {owner}/{repo} --json number,title,body,headRefName,baseRefName,commits

# 変更ファイル一覧とdiffの取得
gh pr diff {pr_number} --repo {owner}/{repo}
```

2. 差分に基づいてレビューを実施

3. レビュー結果を **Pending状態（Start a Review）** で投稿:

**🚨 大原則: 個別の指摘は必ず該当行への inline comment として投稿する**

サマリー本文 (`body`) に指摘を箇条書きで詰め込んではいけない。指摘 1 件 = `comments` 配列の 1 要素として、`path` + `line` を指定して該当コード行に紐付ける。`body` は全体評価 1〜2 文 + 「各論は inline で N 件残します」程度に留める。

理由: GitHub UI 上で該当コードの横にコメントが表示されないと、レビュイーが「どの行の話か」を読み解く負担が増え、スレッドでの議論も成立しない。サマリーに全部書く形は walk-through 記事であってコードレビューではない。

```bash
# stdin経由でJSONを渡して投稿（ファイル書き込み不要）
gh api repos/{owner}/{repo}/pulls/{pull_number}/reviews \
  --method POST \
  --input - <<'EOF'
{
  "commit_id": "<PR の head SHA>",
  "body": "全体評価を 1〜2 文。各論は inline で N 件残します。",
  "comments": [
    {
      "path": "src/example.tsx",
      "line": 42,
      "body": "🔴 **[MUST]** 問題の説明。\n\n**提案**: コード例やアドバイス"
    },
    {
      "path": "src/other.ts",
      "line": 17,
      "body": "🟡 **[SHOULD]** ..."
    }
  ]
}
EOF
```

**重要**:
- **指摘は必ず inline (`comments[]`) として、該当コード行に紐付ける**。サマリー (`body`) に MUST/SHOULD 指摘を書き並べないこと。同じファイルの指摘でも、行が違えば別の inline comment として分ける。
- `commit_id` には PR の最新 head SHA を入れる (`gh api repos/{owner}/{repo}/pulls/{pr_number} --jq '.head.sha'` で取得)。これが無いと「outdated」扱いになりうる。
- `line` にはファイル内の **新ファイル側の行番号** を指定する (diff の position ではない)。`gh api repos/{owner}/{repo}/pulls/{pr_number}/files --jq '.[] | {filename, patch}'` で patch の `@@ -A,B +C,D @@` の C を起点に数える。
- `event` フィールドは **絶対に含めない** こと。省略するとデフォルトで `PENDING`（ドラフト）状態になる。`event` に `"COMMENT"` や `"PENDING"` などを指定すると即座に投稿されてしまうので注意。
- `/tmp` へのファイル書き込みは権限で失敗する可能性があるため、必ず `--input -` でstdin経由のヒアドキュメントを使うこと。
- MCP GitHub ツール (`mcp__github__*`) は使用しないこと。`gh` CLI に統一する。

4. 投稿完了後、PRのURLをユーザーに返し、Pending状態であることを伝える。投稿した inline comment の件数 (例: 「inline 4 件 / サマリー 1 件」) も併せて伝えると、ユーザーが意図通りの形式かを即確認できる。

### owner/repo の解決

```bash
# リモートURLから取得
gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"'
```

- PR URLが渡された場合はURLからパース
- PR番号のみの場合は上記コマンドで現在のリポジトリから取得

### 注意事項

- 引数なしで呼ばれた場合は、ローカル出力のみ行いGitHubへの投稿は行わない
- 既存の Pending review が同じ author で残っている場合は `gh api -X DELETE repos/{owner}/{repo}/pulls/{pr_number}/reviews/{review_id}` で削除してから新規作成する (重複投稿防止)

## レビュー時の心構え

- **批判ではなく改善提案** — "〜は間違い"ではなく"〜するとより良い"
- **コードを批判、人を批判しない** — "このコードは"と書く
- **学びの機会を提供** — なぜそうすべきかの理由を添える
