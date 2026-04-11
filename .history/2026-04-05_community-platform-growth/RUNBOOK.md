# 新プラットフォーム導入ランブック

**作成日:** 2026-04-11
**対象バージョン:** everything-claude-code v1.10.0
**対象者:** ECC を Kiro, CodeBuddy, Trae に導入するユーザー

---

## 目次

1. [前提条件](#1-前提条件)
2. [Kiro IDE への導入](#2-kiro-ide-への導入)
3. [CodeBuddy (Tencent) への導入](#3-codebuddy-tencent-への導入)
4. [Trae IDE への導入](#4-trae-ide-への導入)
5. [選択的インストールによるカスタマイズ](#5-選択的インストールによるカスタマイズ)
6. [トラブルシューティング](#6-トラブルシューティング)

---

## 1. 前提条件

すべてのプラットフォームに共通する前提条件:

- **Node.js 18 以上**がインストールされていること
- **Git** がインストールされ、リポジトリがクローンされていること
- 対象 IDE がインストールされていること

```bash
# リポジトリのクローン
git clone https://github.com/affaan-m/everything-claude-code.git
cd everything-claude-code
```

---

## 2. Kiro IDE への導入

### 2.1 インストール

```bash
# Kiro 用インストーラを実行
bash .kiro/install.sh
```

インストーラは非破壊的に動作し、既存の `.kiro/` 設定を上書きしない。以下がコピーされる:

- 17 エージェント（JSON + Markdown ペア）
- 18 スキル（SKILL.md）
- 10 Hook（`.kiro.hook`）
- Steering 設定

### 2.2 確認

```bash
# エージェントの確認
ls .kiro/agents/*.json | wc -l
# 期待値: 17

# スキルの確認
ls .kiro/skills/*/SKILL.md | wc -l
# 期待値: 18

# Hook の確認
ls .kiro/hooks/*.kiro.hook | wc -l
# 期待値: 10
```

### 2.3 含まれるエージェント

architect, build-error-resolver, code-reviewer, cpp-reviewer, cpp-build-resolver, csharp-reviewer, dart-build-resolver, database-reviewer, doc-updater, e2e-runner, flutter-reviewer, go-reviewer, go-build-resolver, harness-optimizer, java-reviewer, java-build-resolver, kotlin-reviewer, kotlin-build-resolver, loop-operator, planner, python-reviewer, refactor-cleaner, rust-reviewer, rust-build-resolver, security-reviewer, tdd-guide, typescript-reviewer

### 2.4 含まれるスキル

agentic-engineering, api-design, backend-patterns, coding-standards, database-migrations, deployment-patterns, docker-patterns, e2e-testing, frontend-patterns, golang-patterns, golang-testing, postgres-patterns, python-patterns, python-testing, search-first, security-review, tdd-workflow, verification-loop

---

## 3. CodeBuddy (Tencent) への導入

### 3.1 Bash インストール

```bash
# CodeBuddy 用インストーラを実行
bash .codebuddy/install.sh
```

### 3.2 Node.js インストール（代替）

```bash
# Node.js ベースのインストーラ
node .codebuddy/install.js
```

### 3.3 選択的インストール（推奨）

```bash
# 選択的インストールでターゲット指定
./install.sh --target codebuddy --profile standard
```

### 3.4 CN 環境向け

中国環境のユーザーは、中国語ドキュメントを参照:

```bash
cat .codebuddy/README.zh-CN.md
```

### 3.5 アンインストール

```bash
# Bash アンインストーラ
bash .codebuddy/uninstall.sh

# または Node.js アンインストーラ
node .codebuddy/uninstall.js
```

マニフェスト追跡（`ecc-install-state.json`）により、ECC がインストールしたファイルのみが削除される。

---

## 4. Trae IDE への導入

### 4.1 ローカルインストール（プロジェクト内）

```bash
# プロジェクトディレクトリに .trae/ を作成
bash .trae/install.sh
```

### 4.2 グローバルインストール

```bash
# ~/.trae/ にグローバルインストール
bash .trae/install.sh --global
```

### 4.3 CN 環境向け

```bash
# CN 環境では .trae-cn/ が使われる
TRAE_ENV=cn bash .trae/install.sh
```

### 4.4 アンインストール

```bash
bash .trae/uninstall.sh
```

---

## 5. 選択的インストールによるカスタマイズ

v1.9.0 以降、ECC は選択的インストールをサポートしている。各プラットフォームに対して、必要なコンポーネントだけを選択してインストールできる。

### 5.1 プロファイル

```bash
# minimal: ルールとコマンドのみ
./install.sh --profile minimal --target kiro

# standard: ルール + コマンド + エージェント + 基本スキル
./install.sh --profile standard --target codebuddy

# full: 全コンポーネント
./install.sh --profile full --target trae
```

### 5.2 言語の選択

```bash
# 特定の言語のみを追加
./install.sh --with lang:python --with lang:typescript --target kiro

# 特定の言語を除外
./install.sh --profile standard --without lang:java --target codebuddy
```

### 5.3 利用可能なターゲット

| ターゲット | フラグ値 |
|-----------|---------|
| Claude Code | `claude` |
| Cursor | `cursor` |
| Kiro | `kiro` |
| CodeBuddy | `codebuddy` |
| Trae | `trae` |
| Codex | `codex` |
| OpenCode | `opencode` |
| Antigravity | `antigravity` |
| Gemini | `gemini` |

---

## 6. トラブルシューティング

### 6.1 インストーラが失敗する

```bash
# npm 依存関係をインストール
npm install

# 再度インストーラを実行
bash .kiro/install.sh
```

### 6.2 Hook が動作しない

Hook は IDE 固有のイベントシステムに依存する。各 IDE の Hook 設定が有効になっていることを確認:

- **Kiro**: `.kiro/hooks/` 内の `.kiro.hook` ファイルが存在すること
- **CodeBuddy**: ECC のインストール後に IDE を再起動
- **Trae**: `.trae/` 内の設定ファイルが正しく配置されていること

### 6.3 既存の設定との競合

すべてのインストーラは非破壊的に設計されているが、既存の設定と競合が発生した場合:

```bash
# バックアップを取ってからクリーンインストール
cp -r .kiro .kiro.backup
bash .kiro/install.sh --force
```

### 6.4 中国環境での接続問題

CodeBuddy と Trae の CN 環境では、npm レジストリの設定が必要な場合がある:

```bash
npm config set registry https://registry.npmmirror.com
```
