# ECC 2.0 制御プレーン ランブック

**作成日:** 2026-04-11
**対象バージョン:** everything-claude-code v1.10.0（ecc2/ Alpha）
**対象者:** ECC 2.0 をローカルで試したい開発者・オペレーター

---

## 目次

1. [前提条件](#1-前提条件)
2. [ビルドと起動](#2-ビルドと起動)
3. [設定ファイル（ecc2.toml）](#3-設定ファイルecc2toml)
4. [セッション操作](#4-セッション操作)
5. [デリゲーションとチーム管理](#5-デリゲーションとチーム管理)
6. [Worktree 管理](#6-worktree-管理)
7. [コンテキストグラフ](#7-コンテキストグラフ)
8. [定期タスクとリモートディスパッチ](#8-定期タスクとリモートディスパッチ)
9. [TUI ダッシュボード操作](#9-tui-ダッシュボード操作)
10. [レガシーマイグレーション](#10-レガシーマイグレーション)
11. [トラブルシューティング](#11-トラブルシューティング)

---

## 1. 前提条件

- **Rust ツールチェーン**: `rustup` でインストール（edition 2021 以上）
- **Git**: worktree 機能で使用
- **AI ハーネス CLI**: セッション実行に必要（`claude`, `codex`, `gemini-cli` のいずれか）

```bash
# Rust がまだ入っていない場合
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 確認
rustc --version   # 1.70+ を推奨
cargo --version
```

SQLite は `rusqlite` の bundled フィーチャーで同梱されるため、別途インストール不要。

---

## 2. ビルドと起動

### 2.1 ソースからビルド

```bash
cd everything-claude-code/ecc2

# デバッグビルド（初回は依存関係のダウンロードで数分かかる）
cargo build

# リリースビルド（LTO 有効、実行速度が速い）
cargo build --release
```

### 2.2 直接実行

```bash
# cargo run 経由（ビルド + 実行）
cargo run -- dashboard

# リリースビルドのバイナリを直接実行
./target/release/ecc-tui dashboard
```

### 2.3 テスト

```bash
cargo test
```

### 2.4 PATH に追加（任意）

頻繁に使う場合はシンボリックリンクを作成:

```bash
ln -s "$(pwd)/target/release/ecc-tui" ~/.local/bin/ecc
# 以降 ecc dashboard, ecc start ... で起動可能
```

---

## 3. 設定ファイル（ecc2.toml）

ECC 2.0 は TOML 設定を**レイヤー化して読み込む**。下位のファイルが上位を上書きする:

| 優先度 | パス | 用途 |
|-------|------|------|
| 1（低） | `~/.claude/ecc2.toml`（レガシー） | グローバルデフォルト |
| 2 | `$XDG_CONFIG_HOME/ecc2/config.toml` | グローバル設定 |
| 3 | `<project>/.claude/ecc2.toml`（レガシー） | プロジェクト固有 |
| 4（高） | `<project>/ecc2.toml` | プロジェクト固有（推奨） |

### 3.1 最小構成

設定ファイルがなくてもデフォルト値で動作する。最小限のカスタマイズ例:

```toml
# ecc2.toml
default_agent = "claude"

[harness_runners.claude]
program = "claude"
task_flag = "--task"
```

### 3.2 フル構成の例

```toml
# === 基本設定 ===
default_agent = "claude"
default_agent_profile = "standard"
auto_create_worktrees = true
worktree_branch_prefix = "ecc"
max_parallel_sessions = 4

# === ハーネスランナー ===
[harness_runners.claude]
program = "claude"
base_args = []
cwd_flag = "--cwd"
session_name_flag = "--session"
task_flag = "--task"
model_flag = "--model"
allowed_tools_flag = "--allowed-tools"
max_budget_usd_flag = "--max-budget"
append_system_prompt_flag = "--append-prompt"

[harness_runners.codex]
program = "codex"
task_flag = "--task"
model_flag = "--model"

[harness_runners.gemini]
program = "gemini-cli"
task_flag = "--task"

# === エージェントプロファイル ===
[agent_profiles.standard]
agent = "claude"
model = "claude-sonnet-4-5"
allowed_tools = ["bash", "write", "read", "edit", "grep", "glob"]
max_budget_usd = 5.0

[agent_profiles.researcher]
inherits = "standard"
allowed_tools = ["bash", "read", "grep", "web_search", "web_fetch"]
max_budget_usd = 10.0

[agent_profiles.heavy]
agent = "claude"
model = "claude-opus-4-5"
max_budget_usd = 20.0

# === オーケストレーションテンプレート ===
[orchestration_templates.review_and_fix]
description = "Review code and fix issues"
steps = [
    { name = "review", task = "Review {target} for bugs and style issues", profile = "standard" },
    { name = "fix", task = "Fix the issues found in the review", profile = "standard", worktree = true },
    { name = "test", task = "Run tests and verify fixes", profile = "standard" },
]

# === メモリコネクタ ===
[memory_connectors.project_docs]
kind = "markdown_directory"
path = "docs/"
recurse = true
default_entity_type = "documentation"

[memory_connectors.env]
kind = "dotenv_file"
path = ".env.example"
include_safe_values = true
exclude_keys = ["*SECRET*", "*PASSWORD*", "*TOKEN*"]

# === 予算アラート ===
[budget_alert_thresholds]
advisory = 0.50
warning = 0.75
critical = 0.90

# === 通知 ===
[notifications]
desktop = true
webhook_url = ""
```

---

## 4. セッション操作

### 4.1 セッション作成

```bash
# 基本的なセッション開始
ecc start --task "Refactor the auth module for better testability"

# エージェントとプロファイルを指定
ecc start --task "Write E2E tests for login flow" --agent claude --profile researcher

# Worktree 付きで開始（独立ブランチで作業）
ecc start --task "Migrate database schema" -w
```

### 4.2 セッション確認

```bash
# アクティブセッション一覧
ecc sessions

# 特定セッションの詳細（latest で最新）
ecc status latest
ecc status <session-id>
```

### 4.3 セッション停止・再開

```bash
# 停止
ecc stop <session-id>

# 再開（Failed / Stopped 状態のセッション）
ecc resume <session-id>
```

### 4.4 判断ログ

```bash
# 設計判断を記録
ecc log-decision \
  --decision "Use opaque tokens instead of JWT" \
  --reasoning "JWT revocation requires a blacklist; opaque tokens can be revoked instantly" \
  --alternative "Short-lived JWTs with 5-minute expiry"

# 判断ログを閲覧
ecc decisions
ecc decisions --all --limit 50
```

---

## 5. デリゲーションとチーム管理

### 5.1 タスクの委譲

```bash
# リードセッションから明示的に委譲
ecc delegate <lead-session-id> --task "Implement the frontend changes" -w

# スマートアサイン（空きデリゲートがあれば再利用）
ecc assign <lead-session-id> --task "Write unit tests for auth module"
```

### 5.2 チームの確認

```bash
# デリゲーションツリーの表示
ecc team <lead-session-id>

# 深さを指定（デフォルト: 2）
ecc team <lead-session-id> --depth 3
```

### 5.3 メッセージング

```bash
# タスクハンドオフを送信
ecc messages send \
  --from <lead-id> --to <delegate-id> \
  --kind handoff \
  --text "Please review the migration script" \
  --priority high

# 受信箱の確認
ecc messages inbox <session-id>
```

### 5.4 バックログ管理

```bash
# 受信箱のタスクをデリゲートにルーティング
ecc drain-inbox <lead-session-id>

# 全リードの受信箱を一括処理
ecc auto-dispatch

# ディスパッチ + リバランスの統合パス
ecc coordinate-backlog

# バックログが healthy になるまで繰り返す
ecc coordinate-backlog --until-healthy

# チーム内のリバランス
ecc rebalance-team <lead-session-id>

# 全チームのリバランス
ecc rebalance-all

# オーケストレーション状態の確認
ecc coordination-status
ecc coordination-status --json   # 機械可読
ecc coordination-status --check  # 非ゼロ exit code で異常通知
```

### 5.5 テンプレート実行

```bash
# 定義済みテンプレートの実行
ecc template review_and_fix --var target=src/auth/

# リードセッションからテンプレートを実行
ecc template review_and_fix --from-session <lead-id> --var target=src/api/
```

---

## 6. Worktree 管理

### 6.1 状態確認

```bash
# 特定セッションの worktree
ecc worktree-status <session-id>

# 全セッションの worktree
ecc worktree-status --all

# パッチプレビュー付き
ecc worktree-status <session-id> --patch

# JSON 出力
ecc worktree-status --all --json
```

### 6.2 マージ

```bash
# 単一 worktree をベースブランチにマージ
ecc merge-worktree <session-id>

# 全 Ready worktree を一括マージ
ecc merge-worktree --all

# マージ後も worktree を保持
ecc merge-worktree <session-id> --keep-worktree
```

### 6.3 マージキュー

```bash
# マージ待ち行列の確認
ecc merge-queue

# キューを処理（auto-rebase + merge）
ecc merge-queue --apply
```

### 6.4 コンフリクト解決

```bash
# コンフリクトプロトコルの表示
ecc worktree-resolution <session-id>

# 全コンフリクトの確認
ecc worktree-resolution --all
```

### 6.5 プルーニング

```bash
# 不要 worktree の削除
ecc prune-worktrees
```

---

## 7. コンテキストグラフ

### 7.1 エンティティ操作

```bash
# エンティティ追加
ecc graph add-entity \
  --type module \
  --name "auth_service" \
  --path "src/auth/" \
  --summary "Authentication and authorization service" \
  --meta "language=typescript" \
  --meta "framework=express"

# エンティティ一覧
ecc graph entities
ecc graph entities --type file --limit 50

# エンティティ詳細（関係付き）
ecc graph show <entity-id>
```

### 7.2 関係と観察

```bash
# 関係の追加
ecc graph link --from 1 --to 2 --relation depends_on --summary "Auth depends on User model"

# 関係一覧
ecc graph relations --entity-id 1

# 観察の追加
ecc graph add-observation \
  --entity-id 1 \
  --type bug_fix \
  --priority high \
  --pinned \
  --summary "Fixed race condition in token refresh" \
  --detail "root_cause=missing mutex on refresh counter"

# 観察一覧
ecc graph observations --entity-id 1

# ピン操作
ecc graph pin-observation --observation-id 5
ecc graph unpin-observation --observation-id 5
```

### 7.3 リコールと同期

```bash
# キーワードでリコール
ecc graph recall "authentication token refresh"

# セッションの活動からグラフを自動構築
ecc graph sync
ecc graph sync --all

# 圧縮（エンティティあたり12件まで保持）
ecc graph compact
ecc graph compact --keep-observations-per-entity 20
```

### 7.4 メモリコネクタ

```bash
# コネクタの状態確認
ecc graph connectors

# 特定コネクタの同期
ecc graph connector-sync project_docs

# 全コネクタを同期
ecc graph connector-sync --all
```

---

## 8. 定期タスクとリモートディスパッチ

### 8.1 スケジュール管理

```bash
# 定期タスクの追加（平日 9:00 に実行）
ecc schedule add \
  --cron "0 9 * * MON-FRI" \
  --task "Run daily code review on main branch" \
  --agent claude \
  --profile standard

# 一覧
ecc schedule list

# 削除
ecc schedule remove <schedule-id>

# due のスケジュールを今すぐ実行
ecc schedule run-due
```

### 8.2 リモートディスパッチ

```bash
# リモートタスクをキューに追加
ecc remote add --task "Fix the flaky test in CI" --priority high

# Computer Use タスク
ecc remote computer-use \
  --goal "Fill out the form on the internal dashboard" \
  --target-url "https://internal.example.com/form"

# キューの確認
ecc remote list

# キューを処理
ecc remote run

# HTTP エンドポイントとして待ち受け
ecc remote serve --bind 127.0.0.1:8787 --token "my-secret-token"
```

リモートサーブ起動中は `POST /dispatch` に Bearer トークン付きでリクエストを送信できる。

---

## 9. TUI ダッシュボード操作

```bash
ecc dashboard
```

### 9.1 主要キーバインド

| キー | 操作 |
|------|------|
| `Tab` / `Shift+Tab` | ペイン間移動 |
| `[` / `]` | セッション選択 |
| `j` / `k` | スクロール |
| `n` | 新規セッション |
| `N` | 自然言語でセッション作成 |
| `s` / `u` | 停止 / 再開 |
| `m` / `M` | マージ / 全マージ |
| `v` | 出力モード切替（Output → Diff → Git Status → Patch） |
| `K` | コンテキストグラフモード |
| `y` | タイムラインモード |
| `/` | 検索 |
| `T` | テーマ切替 |
| `?` | ヘルプ |
| `q` | 終了 |

### 9.2 オーケストレーション操作

| キー | 操作 |
|------|------|
| `a` | タスク割り当て |
| `g` / `G` | ディスパッチ / 調整 |
| `b` / `B` | チームリバランス / 全リバランス |
| `i` | 受信箱ドレイン |
| `d` | セッション削除 |

---

## 10. レガシーマイグレーション

Hermes / OpenClaw ワークスペースから ECC 2.0 への移行:

```bash
# Step 1: 監査（何があるか確認）
ecc migrate audit --source /path/to/legacy-workspace

# Step 2: 計画（移行ロードマップ生成）
ecc migrate plan --source /path/to/legacy-workspace

# Step 3: 足場作成（config.toml のスケルトン等）
ecc migrate scaffold --source /path/to/legacy-workspace --output-dir ./ecc2-migration/

# Step 4-10: 個別インポート（必要なものだけ選択）
ecc migrate import-schedules --source /path/to/legacy-workspace --dry-run
ecc migrate import-memory --source /path/to/legacy-workspace --limit 200
ecc migrate import-env --source /path/to/legacy-workspace --dry-run
ecc migrate import-skills --source /path/to/legacy-workspace --output-dir ./ecc2-migration/
ecc migrate import-tools --source /path/to/legacy-workspace --output-dir ./ecc2-migration/
ecc migrate import-plugins --source /path/to/legacy-workspace --output-dir ./ecc2-migration/
ecc migrate import-remote --source /path/to/legacy-workspace --dry-run
```

すべてのインポートコマンドに `--dry-run` または `--json` オプションがある。本番実行前に必ずドライランで確認すること。

---

## 11. トラブルシューティング

### 11.1 ビルドが失敗する

```bash
# Rust ツールチェーンを最新に更新
rustup update

# 依存関係のキャッシュをクリア
cargo clean && cargo build
```

### 11.2 データベースの破損

SQLite データベースのパスはデフォルトで `$XDG_CONFIG_HOME/ecc2/ecc2.db`。破損した場合:

```bash
# データベースの場所を確認
ls ~/.config/ecc2/ecc2.db  # Linux
ls ~/Library/Application\ Support/ecc2/ecc2.db  # macOS

# バックアップを取ってから削除（全セッション情報が失われる）
cp ~/.config/ecc2/ecc2.db ~/.config/ecc2/ecc2.db.bak
rm ~/.config/ecc2/ecc2.db
# 次回起動時に自動再作成される
```

### 11.3 セッションが Stale のまま

Daemon がハートビートを検出できず Stale になった場合:

```bash
# セッションの状態確認
ecc status <session-id>

# 手動で停止して再開
ecc stop <session-id>
ecc resume <session-id>
```

### 11.4 Worktree のコンフリクト

```bash
# コンフリクトの詳細を確認
ecc worktree-resolution <session-id>

# 手動で解決する場合は worktree ディレクトリに移動
cd <worktree-path>
git status
# コンフリクトを解決後、ecc merge-worktree で再試行
```

### 11.5 Daemon が起動しない

```bash
# フォアグラウンドで起動してログを確認
RUST_LOG=debug ecc daemon
```

### 11.6 テレメトリのエクスポート

問題調査のためにセッション情報を OTLP 形式でエクスポートできる:

```bash
ecc export-otel > sessions.json
ecc export-otel <session-id> --output single-session.json
```
