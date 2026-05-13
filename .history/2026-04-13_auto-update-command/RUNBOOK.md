# `/auto-update` ランブック

**作成日:** 2026-05-13
**対象バージョン:** everything-claude-code post-v1.10.0
**対象者:** 既に ECC を `/install` 済みで、upstream の更新を取り込みたいユーザー

---

## 目次

1. [前提条件](#1-前提条件)
2. [必須 vs 任意の作業](#2-必須-vs-任意の作業)
3. [基本フロー（dry-run → 適用）](#3-基本フロー)
4. [シナリオ別の使い分け](#4-シナリオ別の使い分け)
5. [トラブルシューティング](#5-トラブルシューティング)

---

## 1. 前提条件

- ECC を 1 度以上 `/install` 済みであること（install-state が SQLite に記録されている）
- `~/.claude/plugins/...` または `$CLAUDE_PLUGIN_ROOT` に ECC ディレクトリが存在すること
- upstream リポジトリへの read アクセス（標準的には GitHub への https 接続）
- Node.js（ECC を含む環境では基本的にインストール済み）

`/auto-update` は git pull → install-apply の再実行という流れを取るため、internet 接続が必要。Offline 環境では `--repo-root` を local mirror に向けることで一部代替できる（ただし pull はスキップされる）。

---

## 2. 必須 vs 任意の作業

| 項目 | 必須/任意 | 補足 |
|------|----------|------|
| ECC 初回 `/install` の完了 | **必須** | install-state がなければ auto-update は対象を発見できない |
| `git pull` を手動で実行 | **不要** | auto-update が内部で実行する |
| `--target` の指定 | 任意 | 省略時は記録されている全 target を更新 |
| `--repo-root` の指定 | 任意 | 省略時は install-state から逆算 |
| `--dry-run` の事前実行 | **強く推奨** | 実 mutation 前に計画を確認できる |
| `~/.claude/settings.json` の手動編集 | **不要** | auto-update が install plan に従って書き換える |

重要：**`/auto-update` の前に `~/.claude/settings.json` を手動で編集する必要はない。** 古いドキュメントで「先に手で `git pull` してから...」と書かれていることがあるが、現バージョンでは不要。

---

## 3. 基本フロー

### Step 1: dry-run で計画を確認

```bash
# 環境変数 ECC_ROOT が解決されている前提（/auto-update の中ではこの解決が自動）
node "$ECC_ROOT/scripts/auto-update.js" --dry-run
```

または Claude Code 内で:

```
/auto-update --dry-run
```

出力例:

```
[auto-update] Discovered install-states: 2 (target=claude, target=cursor)
[auto-update] Repo root: /Users/you/.claude/plugins/marketplace/everything-claude-code
[auto-update] Would git pull origin main
[auto-update] Plan for target=claude:
  - copy skills/dependency-audit/SKILL.md → ~/.claude/skills/...
  - update rules/node.md → ~/.claude/rules/node.md
  - remove skills/legacy-foo/SKILL.md (upstream で削除)
[auto-update] Plan for target=cursor:
  - update .cursor/rules/coding-style.md
[auto-update] DRY RUN. No mutation performed.
```

差分が想定通りなら次へ。

### Step 2: 実行

```
/auto-update
```

`--dry-run` を外すと、auto-update が次を実行する。

1. `git pull --ff-only` を `--repo-root` で取得
2. 各 target について `install-apply.js` を recorded request で再実行
3. install-state を最新の operations で上書き

### Step 3: 確認

```
/install-status
```

これで各 target の最新 install 結果を確認できる。Claude Code を再起動して、追加された skill が `/<skill-name>` で呼べることを確かめる。

---

## 4. シナリオ別の使い分け

### Cursor だけ最新化したい

```
/auto-update --target cursor
```

Claude plugin 側はそのまま、Cursor 向けファイルだけ更新する。

### 別の repo クローンを使う

```
/auto-update --repo-root /Users/you/dev/everything-claude-code
```

社内 fork や検証用 branch を持っているとき、そちらを source にする。

### 結果を CI で読みたい

```
/auto-update --json --dry-run > plan.json
```

`--json` を付けると、出力が機械可読 JSON になる。CI で「変更がある場合のみ Slack 通知」のような条件を書くときに使う。

---

## 5. トラブルシューティング

### `scripts/auto-update.js: not found`

npm 経由で ECC を入れた場合、古いバージョンでは script が npm package に同梱されていなかった。`2006d2ee` 以降のバージョンで解消。`npm update ecc-universal` で最新化すれば解決する。

### `git pull` で merge conflict

ECC リポジトリ側で local 変更があるとき発生する。`/auto-update` は `--ff-only` で pull するため、conflict 時は中断する。local 変更を別 branch に移すか stash してから再実行する。auto-update 自体は破壊的操作を行わずに終了する。

### `Could not derive repo root from install-state`

install-state の `operations` に `sourcePath` 情報が欠けているケース。古い install で発生することがある。

```
/auto-update --repo-root /Users/you/.claude/plugins/...
```

と明示的に渡せば解決する。次回以降は install-state が補完される。

### `install-apply.js` が一部 target で失敗

target ごとに独立して実行されるため、cursor が失敗しても claude は更新される。失敗した target だけ再実行できる。

```
/auto-update --target cursor
```

エラーログには failing operation の詳細が出るので、それを見て対処する。
