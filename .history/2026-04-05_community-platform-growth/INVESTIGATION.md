# コミュニティ成長とマルチプラットフォーム対応 調査レポート

**調査日:** 2026-04-11
**対象バージョン:** everything-claude-code v1.10.0（v1.9.0..HEAD、627コミット）
**調査者:** Claude Opus 4.6
**対象領域:** コミュニティ貢献による IDE 対応拡張、新言語サポート、翻訳、および OSS 成長

---

## 1. v1.10.0 で何が変わったのか

v1.9.0 が「選択的インストール」と「言語エコシステム拡張」で ECC の内部構造を刷新したのに対し、v1.10.0 では**外向きの拡張**が顕著である。627コミットのうち約150がコミュニティからの PR マージで占められ、3つの新 IDE 対応（Kiro, CodeBuddy, Trae）、4つの新言語ルール（C#, Dart, C, Web Frontend）、4つの新翻訳（トルコ語、ポルトガル語、日本語強化、中国語拡張）が追加された。

この動きは、ECC が「Claude Code 専用の設定集」から「マルチハーネス対応のエージェント基盤」へと再定義されたことの自然な帰結である。対応ハーネスが増えれば、それぞれのコミュニティからの貢献が集まる。v1.10.0 の公開表面は 38 エージェント、156 スキル、72 コマンドに到達した。

---

## 2. 新 IDE 対応

### 2.1 Kiro IDE（`.kiro/`）

Kiro は、ECC が最も包括的にサポートする Claude Code 以外の IDE である。`.kiro/` ディレクトリに以下が配置されている:

| カテゴリ | ファイル数 | 内容 |
|---------|-----------|------|
| エージェント | 34（17 JSON + 17 MD） | architect, build-error-resolver, code-reviewer, cpp-reviewer, database-reviewer, e2e-runner, flutter-reviewer, go-reviewer, harness-optimizer, java-reviewer, kotlin-reviewer, loop-operator, planner, python-reviewer, refactor-cleaner, rust-reviewer, security-reviewer, tdd-guide, typescript-reviewer |
| スキル | 18 SKILL.md | agentic-engineering, api-design, backend-patterns, coding-standards, database-migrations, deployment-patterns, docker-patterns, e2e-testing, frontend-patterns, golang-patterns, golang-testing, postgres-patterns, python-patterns, python-testing, search-first, security-review, tdd-workflow, verification-loop |
| Hook | 10 `.kiro.hook` | auto-format, code-review-on-write, console-log-check, doc-file-warning, extract-patterns, git-push-review, quality-gate, session-summary, tdd-reminder, typecheck-on-edit |
| インストーラ | 1 | install.sh（非破壊的インストール） |

Kiro のエージェント定義は JSON + Markdown のペア構成を採用しており、JSON がメタデータ（名前、説明、ツール許可リスト、モデル）を、Markdown がシステムプロンプトを保持する。これは Claude Code のエージェント（YAML フロントマター付き Markdown 単体）とは異なるフォーマットだが、内容的には同等のドメイン知識を含んでいる。

Hook は `.kiro.hook` 拡張子を使い、Kiro 固有のイベントシステム（write, edit, push）にバインドされる。ECC の `hooks.json` ベースの仕組みとは独立しているが、実行するロジックは共通のスクリプト群を参照する。

### 2.2 CodeBuddy（Tencent）（`.codebuddy/`）

PR #1038 で追加された Tencent の CodeBuddy IDE への対応である。インストーラは Bash と Node.js の2系統で提供されている。

```
.codebuddy/
├── install.sh        # Bash インストーラ（231行）
├── install.js        # Node.js インストーラ（312行）
├── uninstall.sh      # Bash アンインストーラ（184行）
├── uninstall.js      # Node.js アンインストーラ（291行）
├── README.md         # 英語ドキュメント
└── README.zh-CN.md   # 中国語ドキュメント
```

インストールターゲットアダプタ（`scripts/lib/install-targets/codebuddy-project.js`、47行）が選択的インストールシステムに統合されており、`--target codebuddy` フラグでプロファイル駆動のインストールが可能。インストール状態は `ecc-install-state.json` に永続化される。

### 2.3 Trae IDE（`.trae/`）

PR #985 で追加された Trae IDE（おそらく中国市場向け）への対応。特筆すべきは中国環境（CN）への明示的なサポートで、`TRAE_ENV=cn` 環境変数を設定すると `.trae-cn/` ディレクトリが使われる。

```
.trae/
├── install.sh        # マルチモードインストーラ
├── uninstall.sh      # マニフェストベースのアンインストーラ
├── README.md         # 英語ドキュメント
└── README.zh-CN.md   # 中国語ドキュメント
```

インストーラはローカル（プロジェクト内 `.trae/`）とグローバル（`~/.trae/`）の両モードをサポートし、マニフェスト追跡による安全なアンインストールが可能。プロジェクトルートのエージェント、スキル、コマンド、ルールを再利用する設計になっている。

### 2.4 OpenCode の拡張

OpenCode は v1.9.0 で基本対応が入っていたが、v1.10.0 で大幅に強化された。

PR #726 で11のエージェントプロンプトが追加され、全24エージェント構成に到達した。追加されたのは cpp-reviewer, cpp-build-resolver, java-reviewer, java-build-resolver, kotlin-reviewer, kotlin-build-resolver, python-reviewer, docs-lookup, harness-optimizer, loop-operator の各プロンプトである。

PR #815 では changed-files ツリー機能が追加され、OpenCode のプラグインシステムにファイル変更追跡が統合された。`.opencode/tools/changed-files.ts` でファイル変更を diff インジケータ付きで表示し、`.opencode/plugins/lib/changed-files-store.ts` が状態管理を担う。

---

## 3. 新言語サポート

v1.9.0 で Java, PHP, Perl, Kotlin, C++, Rust の6言語が追加され、対応言語が12に到達していた。v1.10.0 ではさらに4つのカテゴリが追加された。

### 3.1 C#（`rules/csharp/`）

PR #704 で追加。5つのルールファイルで構成される:

| ファイル | 内容 |
|---------|------|
| `coding-style.md` | .NET 命名規則、async/await パターン、LINQ の使い方 |
| `hooks.md` | StyleCop アナライザー、null 安全性チェック |
| `patterns.md` | 依存性注入、Entity Framework パターン |
| `security.md` | OWASP パターン、認証/認可 |
| `testing.md` | NUnit/xUnit パターン、モック戦略 |

### 3.2 Dart（`rules/dart/`）

C# と同じコミット（`badccc3`）で追加。Flutter を含む Dart エコシステム全体をカバーする。

| ファイル | 行数 | 内容 |
|---------|------|------|
| `coding-style.md` | 159 | Dart イディオム、null 安全、フォーマット |
| `hooks.md` | 66 | アナライザールール、フォーマットチェック |
| `patterns.md` | 261 | 状態管理、Widget パターン |
| `security.md` | 135 | 入力バリデーション、暗号化パターン |
| `testing.md` | 215 | ユニット/Widget/統合テストパターン |

別途、PR #716 で flutter-reviewer エージェントと dart-flutter-patterns スキルが追加され、Flutter 開発のエージェント支援が実現した。

### 3.3 Web Frontend（`rules/web/`）

`31c9f7c` で追加。HTML/CSS/JavaScript の汎用的な Web フロントエンドルールと、デザイン品質チェック用の Hook が含まれる。

| ファイル | 内容 |
|---------|------|
| `coding-style.md` | HTML/CSS/JS 規約、アクセシビリティ |
| `design-quality.md` | UI/UX 原則、レスポンシブデザイン |
| `hooks.md` | リンティング、アクセシビリティチェック、デザイン品質 |
| `patterns.md` | コンポーネントパターン、状態管理 |
| `performance.md` | バンドル最適化、Core Web Vitals |
| `security.md` | CSP、XSS 対策、セキュアヘッダー |
| `testing.md` | E2E、ユニット、ビジュアルリグレッションテスト |

デザイン品質に特化した Hook（`design-quality-check.js`）がこのルールセットに連動しており、フロントエンド編集時にジェネリックな UI パターン（"Get Started" のような缶詰コピー、ストックグラデーション、デフォルトフォント）を検出して警告する仕組みが導入された。

### 3.4 C 言語互換性

`86cbe3d` で追加。専用の `rules/c/` ディレクトリはまだ作成されていないが、インストーラの言語エイリアスマップに C が追加され、選択的インストールで `--with lang:c` が使えるようになった。

---

## 4. 翻訳の拡張

### 4.1 トルコ語（tr）

PR #744 で 26,670 行の包括的なトルコ語ドキュメントが追加された。37以上のエージェント翻訳、30以上のコマンドドキュメント、40以上のスキルドキュメント、ルール翻訳（common, golang, python, typescript）、コアドキュメント（CONTRIBUTING.md, CHANGELOG.md, TROUBLESHOOTING.md 等）を含む。

### 4.2 ポルトガル語（pt-BR）

PR #736 で 6,059 行のブラジルポルトガル語ドキュメントが追加された。16以上のエージェント翻訳、13以上のコマンドドキュメント、5以上のルール翻訳、コアドキュメントとサンプルを含む。

### 4.3 日本語（ja-JP）の強化

PR #897 で22ファイルのプレーンテキストコードブロック翻訳が追加された。コードサンプル内の説明テキストが日本語化され、ドキュメントの一貫性が向上した。

### 4.4 中国語（zh-CN）の拡張

新しい言語ルール（C#, C++）の中国語翻訳が追加され、共通ルールとコマンドの更新も行われた。PR #728 で最新のアップストリーム変更との同期が完了した。

---

## 5. 主要コミュニティ PR 一覧

v1.10.0 に含まれるコミュニティ PR の中で特に影響の大きいものを挙げる。

| PR | 著者 | 変更内容 | 影響度 |
|----|------|---------|--------|
| #716 | Maciej Starosielec | Flutter reviewer エージェントと skill | 新エコシステム対応 |
| #726 | nayanjaiswal1 | OpenCode エージェント11個追加 | OpenCode 完全対応 |
| #736 | pvgomes | ブラジルポルトガル語翻訳 | 新言語対応 |
| #744 | Berkcan Gumusisik | トルコ語翻訳（26,670行） | 新言語対応 |
| #815 | Neha Prasad | OpenCode changed-files ツリー | 機能拡張 |
| #848 | 不明 | PRP ワークフローコマンド | 新ワークフロー |
| #897 | techiro | 日本語ドキュメント強化 | 翻訳品質向上 |
| #985 | likzn | Trae IDE 統合 | 新プラットフォーム |
| #1019 | 不明 | WSL デスクトップ通知 | Windows 対応強化 |
| #1029 | 不明 | GAN スタイル生成/評価ハーネス | 実験的機能 |
| #1034 | 不明 | hexagonal-architecture スキル | アーキテクチャスキル |
| #1036 | 不明 | opensource-pipeline ワークフロー | OSS 公開支援 |
| #1038 | Qingzhou-Joshua | CodeBuddy (Tencent) 対応 | 新プラットフォーム |
| #1052 | 不明 | santa-loop 敵対的レビューコマンド | 品質チェック |

---

## 6. プラグインマニフェストの更新

### 6.1 Claude Plugin（`.claude-plugin/plugin.json`）

バージョン 1.10.0 に更新。38エージェントが登録され、新しい C#, Dart, Flutter のレビューアーが含まれる。キーワードに agents, skills, hooks, TDD, code-review, security が含まれる。

### 6.2 Codex Plugin（`.codex-plugin/plugin.json`）

バージョン 1.10.0 に更新。156の共有 ECC スキル、MCP サーバー設定、TDD/セキュリティ/検証ワークフローのデフォルトプロンプトが含まれる。

### 6.3 Marketplace メタデータ（`.agents/plugins/marketplace.json`）

カテゴリ: Productivity、インストールポリシー: AVAILABLE。

---

## 7. 調査所見のまとめ

### 実装状態の総括

| 機能領域 | 追加量 | 備考 |
|---------|--------|------|
| IDE 対応 | 3 新規（Kiro, CodeBuddy, Trae） | Kiro が最も包括的 |
| 言語ルール | 4 新規（C#, Dart, Web, C） | C はエイリアスのみ |
| 翻訳 | 2 新規（tr, pt-BR） + 2 強化（ja-JP, zh-CN） | tr が最大規模 |
| エージェント | 38 に到達 | v1.9.0 の 27 から増加 |
| スキル | 156 に到達 | v1.9.0 の 113 から増加 |
| コマンド | 72 に到達 | v1.9.0 の 58 から増加 |

### 注目すべき設計判断

1. **非破壊的インストーラ**: すべての新 IDE 対応（Kiro, CodeBuddy, Trae）は非破壊的なインストーラを採用している。既存ファイルを上書きせず、マニフェスト追跡によるクリーンなアンインストールをサポートする。

2. **コンテンツの再利用**: 各 IDE アダプタは、プロジェクトルートのエージェント、スキル、ルールを参照/コピーする設計であり、IDE ごとに独立したコンテンツを維持する必要がない。これにより、新しいスキルやエージェントが追加されたとき、すべての IDE 対応に自動的に反映される。

3. **選択的インストール統合**: CodeBuddy のインストールターゲットアダプタ（`codebuddy-project.js`）が示すように、新 IDE の追加はインストールターゲットアダプタの実装（約50行）と設定の追加で完了する。v1.9.0 の選択的インストールアーキテクチャが功を奏している。

新プラットフォームの導入手順は [RUNBOOK.md](./RUNBOOK.md) を参照。
