---
name: sync-skills
description: dotclaudeリポジトリのskillsと ~/.claude/skills をsymlinkで同期する。新マシンでのセットアップや、dotclaudeにskillを追加/削除した後のリンク張り直しに使う。
disable-model-invocation: true
---

dotclaude を single source of truth とし、`~/.claude/skills/<name>` をリポジトリの
`skills/<name>` への **シンボリックリンク**にして同期を保つ。実体が1つになるため、
どちらのパスから編集しても常に同一になる（drift が原理的に起きない）。

## 重要な前提

- **書き込み先が dotclaude（`~/dev/dotclaude` 配下）になる操作はサンドボックス外**。
  `cp` / `ln` / `git` など dotclaude を変更する Bash は `dangerouslyDisableSandbox: true` で実行する。
- `~/.claude/skills` 配下への symlink 作成はサンドボックス内で可。
- `find-skills` や `grill-me` のような `../../.agents/skills/` を指す symlink は
  **別系統（プラグイン等の管理）なので絶対に触らない**。

## 手順

### 1. パスを解決

dotclaude の場所は環境依存なので、この skill 自身の symlink から逆算する（ハードコードしない）：

```bash
SELF="$(readlink ~/.claude/skills/sync-skills)"   # 例: /Users/<you>/dev/dotclaude/skills/sync-skills
SRC="$(cd "$(dirname "$SELF")" && pwd)"            # = dotclaude/skills
DEST="$HOME/.claude/skills"
echo "SRC=$SRC"; echo "DEST=$DEST"
```

readlink が空（このskillがまだリンクされていない新マシン）の場合は、ユーザーに
dotclaude の clone 先を確認し、`SRC=<clone先>/skills` を使う。

### 2. 現状を点検（変更しない）

```bash
for src in "$SRC"/*/; do
  name="$(basename "$src")"; link="$DEST/$name"
  if [ -L "$link" ] && [ "$(readlink "$link")" = "$SRC/$name" ]; then echo "[ok]       $name"
  elif [ -L "$link" ]; then echo "[relink]   $name (現: $(readlink "$link"))"
  elif [ -e "$link" ]; then echo "[CONFLICT] $name は実体ディレクトリ（dotclaude未取込/driftの恐れ）"
  else echo "[link]     $name (未作成)"; fi
done
# dotclaude管理外（.agents等）は参考表示のみ・変更しない
for d in "$DEST"/*; do
  name="$(basename "$d")"; [ -d "$SRC/$name" ] && continue
  echo "[extern]   $name -> $(readlink "$d" 2>/dev/null || echo 実体)  ※触らない"
done
```

### 3. ユーザーに方針を確認してから反映

点検結果を要約して提示する。`[link]` `[relink]` はそのまま symlink を作成/張り直す。
`[CONFLICT]`（`~/.claude/skills` 側に実体ディレクトリがある）は**勝手に破棄しない**：

- その実体が dotclaude にまだ無い独自skillなら → 「dotclaude に取り込んでから symlink 化」を提案。
- dotclaude にも同名があり内容が違う（drift）なら → `diff` を見せ、**どちらを正にするか**ユーザーに決めてもらう。

合意後に反映（dotclaude を触る行は **サンドボックス無効**で）：

```bash
# link / relink（DEST側のみ。サンドボックス内で可）
rm -f "$DEST/$name" && ln -s "$SRC/$name" "$DEST/$name"

# CONFLICT解消で「実体をdotclaudeへ取り込む」場合（dotclaude書込=サンドボックス無効）
cp -R "$DEST/$name" "$SRC/$name" && rm -rf "$DEST/$name" && ln -s "$SRC/$name" "$DEST/$name"
```

### 4. 検証

```bash
ls -la "$DEST"        # 管理対象が全て -> dotclaude/skills/* の symlink になっているか
```

### 5. 版管理（任意・ユーザーが望めば）

skillの追加/変更を dotclaude にコミットしてバックアップ。ユーザーが明示的に頼んだ時だけ：

```bash
cd ~/dev/dotclaude && git add skills && git status   # 確認してからcommit
```

## 補足

- 双方向コピーではなく **symlink** なので、日常の skill 編集は ~/.claude / dotclaude
  どちらのパスで行っても自動的に同一。明示的な「同期実行」が要るのは
  **skillの新規追加・削除・新マシン初期化・リンク破損時**のみ。
