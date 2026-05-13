# ECC 2.0 を社内で語るための 4 本の記事

**作成日:** 2026-05-13
**対象バージョン:** everything-claude-code v2.0.0-rc.1
**想定読者:** 社内エンジニア（Claude Code / Codex / Cursor を日常的に使う層）
**形式:** 各記事が独立した技術ブログ。1 本あたり 20 分の発表に展開できる粒度

---

## このフォルダの位置づけ

`.history/` の他フォルダは「コミット範囲を絞った調査レポート」だが、このフォルダは目的が違う。社内エンジニア向けに、ECC 2.0 が **誕生から v2.0.0-rc.1 までに何を作り、何を解決したのか** を、20 分の発表 1 回ぶんとして語れる単位に分解した連作記事である。

ECC（Everything Claude Code）は元々 Claude Code 向けのプラグイン集だった。v2.0 で「複数ハーネスを横断する operator 向け control plane」へと舵を切り、Rust 製のネイティブバイナリ `ecc-tui`（約 65,000 行）を中心に据えた。v1.10.0 (2026-04-05) で Alpha、v2.0.0-rc.1 (2026-04-28) で release candidate に到達している。

このフォルダで「ECC2」と呼ぶときは、この `ecc2/` 配下の Rust バイナリと、それを支える skills、cross-harness 規約、Hermes との境界線をすべて含む。Claude Code プラグイン本体（`.claude-plugin/`）とは別物として扱う。

---

## 4 本の記事と重要度

生産性向上への貢献度が大きい順に並べた。発表会で 1 本だけ選ぶなら、上から順番に検討するとよい。

| # | 記事 | 主題 | こんな人に刺さる |
|---|------|------|----------------|
| 01 | [マルチハーネス control plane](./01-multi-harness-control-plane.md) | Claude/Codex/Cursor/OpenCode/Gemini を 1 つの session store と board で束ねる substrate 設計 | 日常で AI を切り替えながら使う人、harness ロックインに違和感がある人 |
| 02 | [Memory・Context Graph](./02-memory-and-context-graph.md) | session 間で知識を持ち回るための SQLite ベースのグラフ記憶層 | `/clear` のたびに背景説明をやり直している人、長期プロジェクトで context が散逸している人 |
| 03 | [Claude Code を拡張する外部脳](./03-extending-claude-code.md) | 並列セッション、長期タスク、scheduled job を可能にする daemon と delegation 機構 | 1 タスク 1 セッション設計の限界を感じている人、夜間バッチで AI を動かしたい人 |
| 04 | [Hermes operator 連携](./04-hermes-operator-integration.md) | 人間オペレーターが使う private workflow を sanitized な公開 skill に変換する境界線設計と board observability | operator 業務を skill 化したい人、複数 session の状況を観測したい人 |

各記事は独立して読めるよう前提を最小限に書き直しているので、最後の 1 本だけ拾い読みしても文意は通る。ただし 01 → 02 → 03 → 04 の順で読むと「control plane → 記憶層 → 並列・長期化 → 人間連携」の積み上げが見えるよう構成してある。

---

## 各記事の構成（共通）

どの記事も Problem → Architecture → Demo の 3 部構成にしている。20 分の発表に当てはめると、おおよそ次の時間配分になる。

| パート | 目安時間 | スライド数 | 内容 |
|--------|---------|-----------|------|
| Problem | 5 分 | 4-5 枚 | なぜ Claude Code 単体ではダメなのか。具体的な痛みのシナリオ |
| Architecture | 8 分 | 6-8 枚 | ECC2 が採用した設計と、選ばれなかった代替案 |
| Demo | 5 分 | 3-4 枚 | 実際の CLI コマンドと TUI 操作。可能ならライブデモ |
| Q&A | 2 分 | - | 質疑応答 |

Mermaid 図はそのままスライドに貼れる粒度で描いているので、発表時はスクリーンショット化するか SVG にして取り込むとよい。

---

## 発表者向けのチートシート

**用語の整理**: 社内 audience が混同しがちな単語を最初に確認しておくと安全。

- **harness**: AI エージェントが動く実行環境。Claude Code、Codex CLI、Cursor、OpenCode、Gemini など
- **session**: ECC2 が SQLite 上で管理する 1 つの agent 実行単位。Claude transcript file とは別概念
- **board**: ECC2 TUI 上の Kanban 風観測面。GitHub Projects ではない
- **operator**: ECC2 が想定する人間ユーザー。エンジニアでもよいし、PM や運用担当でもよい
- **Hermes**: 社外向けには公開しない operator 専用 shell。ECC は Hermes が使う public な workflow 層

**話す前のチェック**:

- ライブデモをやるなら `cargo build --manifest-path ecc2/Cargo.toml` を事前に通しておく
- `cargo run -- dashboard` で TUI が立ち上がることを確認
- 質問対応として、各記事に対応する `.history/` の調査レポートを開けるようにしておく

---

## 関連 .history への参照

このフォルダの記事は、過去の調査レポートを下敷きにしている。深掘り質問への参照用に対応関係を載せる。

| この記事 | 下敷きにした調査レポート |
|---------|-----------------------|
| 01 マルチハーネス | [`2026-04-11_ecc2-memory-harness-migration/`](../2026-04-11_ecc2-memory-harness-migration/INVESTIGATION.md) のマルチハーネスランナー章、[`2026-04-28_ecc2-rc1-release-surface/`](../2026-04-28_ecc2-rc1-release-surface/) (作成中) |
| 02 Memory | [`2026-04-11_ecc2-memory-harness-migration/`](../2026-04-11_ecc2-memory-harness-migration/INVESTIGATION.md) の Context Graph 章 |
| 03 Claude Code 拡張 | [`2026-04-05_ecc2-control-plane-alpha/`](../2026-04-05_ecc2-control-plane-alpha/INVESTIGATION.md) の daemon と delegation 章 |
| 04 Hermes 連携 | [`2026-04-18_ecc2-board-observability/`](../2026-04-18_ecc2-board-observability/INVESTIGATION.md)、`skills/hermes-imports/SKILL.md`、`docs/architecture/cross-harness.md` |
