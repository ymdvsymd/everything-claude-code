# ECC v2.0.0-rc.1 Release Surface 調査レポート

**調査日:** 2026-05-13
**対象バージョン:** everything-claude-code v2.0.0-rc.1（`0a87323e` を中心に `69b8ec4e`, `4b67c3ca`, `5595c074`, `6c8a6bd7`, `530088c7`, `d89f8d89`, `fd95cf6b`, `d2ee1323`, `b6b5b6d0`, `2eaafc38`, `177b8f31`）
**調査者:** Claude Opus 4.7
**対象領域:** v2.0.0-rc.1 release surface（`docs/releases/2.0.0-rc.1/`、`docs/architecture/cross-harness.md`、`skills/hermes-imports/`、release workflow、observer 信号処理）

---

## 1. v2.0.0-rc.1 で何が変わったのか

v2.0.0-rc.1 は ECC を「Claude Code 用のプラグイン集」から「cross-harness operating system for agentic work」へと positioning し直す最初の release candidate である。release notes が明示する通り、Claude Code は依然 core target だが、Codex、OpenCode、Cursor、Gemini は同等の execution surface として扱われる。ECC が reusable substrate で、Hermes はその上に乗る operator shell である、という二層構造を文書化したのが本リリースの核心になる。

`0a87323e`（40 files、863 insertions）が中心 commit で、release surface の主要要素を一括で導入した。`docs/releases/2.0.0-rc.1/` 配下の release notes、social drafts、launch checklist、demo prompts、telegram handoff、quickstart、x thread、`docs/architecture/cross-harness.md` の portability model、そして `skills/hermes-imports/SKILL.md` という import 規約 skill。これに加えて、release workflow、plugin manifest、各 README の翻訳、package metadata の version bump が全部入っている。

周辺の小さな commit がこれを支える。`69b8ec4e` で rc1 quickstart 導線、`4b67c3ca` で release policy drift（テストと docs のずれ）を closure、`5595c074` で install/uninstall README clarity、`6c8a6bd7` と `530088c7` で hook addendum から unicode safety violation marker を除去、`d89f8d89` で Codex 側の skill metadata 正規化、`fd95cf6b` で observer の SIGUSR1 retry、`d2ee1323` で過去の調査レポート 3 本（plugin-install-surface-hardening、CLv2 entrypoint fixes、hook runtime improvements）を `.history/` に追加、`b6b5b6d0` で CI catalog validator のテスト、`2eaafc38` で release workflow YAML の改行正規化、`177b8f31` で install/uninstall path docs の明確化、と続く。

ここでの **release surface** は、ユーザーが ECC をリリース版として認識する全要素を指す。package metadata、README、release notes、launch collateral（X thread、LinkedIn post 等）、quickstart、cross-harness 規約文書、Hermes import skill がすべて含まれる。後で個別に直すのではなく、release candidate という単位で一括整合させた。また、ここでの **Hermes** は ECC とは別の operator 専用 shell を指し、release surface 上は「sanitized setup guide」と「import 規約」だけが公開される。

---

## 2. Release artifacts の集約

### 2.1 `docs/releases/2.0.0-rc.1/` ディレクトリ

`0a87323e` は次のファイル群を `docs/releases/2.0.0-rc.1/` に新規追加した。Release engineering 担当者が異なる surface ごとにバラバラに書いていた launch collateral を、reviewed な surface に集約する設計である。

| ファイル | 内容 |
|---------|------|
| `release-notes.md` | positioning、what changed、why、boundaries、upgrade motion |
| `article-outline.md` | 長文 release post の構造案 |
| `demo-prompts.md` | live demo で使う prompt 集 |
| `launch-checklist.md` | release 当日に踏むチェック項目 |
| `linkedin-post.md` | LinkedIn 用の draft |
| `x-thread.md` | X (Twitter) thread の draft |
| `telegram-handoff.md` | 内部 telegram channel への引き継ぎ note |

これらをすべて repo に commit することで、release が「一人の operator の頭の中」ではなく、reviewable な状態で進行する。後の `publication-evidence-2026-05-12.md` や `publication-evidence-2026-05-13.md` のような事後 evidence もこのディレクトリに追加されていく。

### 2.2 Version bump の範囲

`0a87323e` の中で、次のメタデータ全部が `2.0.0-rc.1` に揃えられた。

- `package.json#version`
- `.claude-plugin/marketplace.json`
- `.codex-plugin/plugin.json`
- `.opencode/package.json` と `package-lock.json`
- `agent.yaml#version`
- `VERSION` ファイル
- `.agents/plugins/marketplace.json`
- 各言語 README（en、zh-CN、pt-BR、tr、zh-TW、ja-JP、ko-KR、ru、vi-VN）

ECC は harness ごとに別の manifest を持つ。すべて同じ semantic version を共有しているのは、ユーザーが「どの harness 経由で入れても同じ snapshot」を期待できる前提を守るためである。version bump は表面的な作業に見えるが、9 種類の翻訳を含めて整合性を取る作業はそれなりの labor coordination を要する。

---

## 3. Cross-harness portability model の文書化

`docs/architecture/cross-harness.md`（111 行）は ECC が初めて「複数 harness にまたがる workflow 層」としての規約を明文化した文書である。詳細は [DEEP-DIVE-CROSS-HARNESS.md](./DEEP-DIVE-CROSS-HARNESS.md) で扱うが、要点は次の通り。

- ECC は reusable substrate。harness はその上の execution surface
- skills、rules、hooks、MCPs、commands は ECC 側に置き、harness 側は adapter のみ
- `SKILL.md` が「最も portable」な単位
- harness 間の差は adapter で吸収。共通 behavior を harness 別 file に複写しない
- Hermes は public ECC runtime ではない。operator shell として ECC asset を消費する側

この文書が `docs/architecture/` 配下にあること自体が意味を持つ。`docs/architecture/` には他にも `harness-adapter-compliance.md`、`observability-readiness.md`、`progress-sync-contract.md`、`evaluator-rag-prototype.md` といった ECC の柱となる contract 文書が並ぶ。Architecture は readme から少し外し、専用 directory で版管理する構造ができてきた。

`tests/docs/ecc2-release-surface.test.js`（120 行）は、これらの release surface 文書が「壊れた link を持たない」「version 数が一致する」「mandatory section が存在する」といった条件を CI で検証する。`4b67c3ca` の「rc1 release policy drift を close」はこのテストの整合修正である。

---

## 4. Hermes imports skill — 規約 skill の導入

`skills/hermes-imports/SKILL.md`（88 行）は、ECC が「他者の workflow を取り込む」ための規約を skill 形式で初めて公開した。詳細は [DEEP-DIVE-HERMES-IMPORTS.md](./DEEP-DIVE-HERMES-IMPORTS.md) で扱う。

ポイントは次の通り。

- import rules（local path を repo-relative に、account 名を role label に、credentials を provider 名だけに）
- sanitization checklist（absolute path、`~/.hermes` 参照、API key、phone number、client name、revenue/health/CRM detail）
- conversion pattern（6 step）
- two example: launch handoff、quiet-hours operator job
- output contract: candidate ECC skill name、sanitized workflow summary、required public inputs、private inputs removed、remaining risks

これは「Hermes 以外の private workflow を ECC に取り込むときの一般規約」として読み替え可能である。社内固有の prompt を OSS skill に変換するときの guideline としても機能する。

`scripts/release.sh` には Hermes import を含むリリース手順との接続が追加されており、release engineering 側の workflow に組み込まれている。

---

## 5. その他の重要な修正

### 5.1 Observer SIGUSR1 retry (`fd95cf6b`)

continuous-learning v2 の `observer-loop.sh` は、SIGUSR1 を受け取ったとき learning material を flush する設計である。RC1 直前のテストで、signal 受信直後の `wait` がすぐに復帰してしまうケースが発見された。Bash の `wait` builtin は signal で interrupt されると非ゼロ exit code で抜けるが、これを「実行終了」と誤判定してループを抜けてしまう問題。

修正は、`wait` を short-circuit ループに包み、SIGUSR1 受信直後は再度 `wait` し直す形に変更している。`tests/hooks/observer-memory.test.js`（+57 行）で signal タイミングのケースを cover した。

```bash
# 概念的な構造
while kill -0 "$pid" 2>/dev/null; do
  wait "$pid" || true
done
```

これは小さな修正だが、continuous learning v2 の reliability を RC1 surface 上で担保するための重要な closure である。

### 5.2 Codex skill metadata 正規化 (`d89f8d89`)

`.agents/skills/*/SKILL.md` には、Codex / OpenAI 向けに `agents/openai.yaml` という manifest が併設されている。RC1 直前の audit で、複数の skill で `SKILL.md` の `tools` frontmatter line がトリミングされていない、あるいは `agents/openai.yaml` が欠落しているといった metadata drift が検出された。

修正は次の 2 種類の機械的整合を行う。

- `SKILL.md` から重複している `tools` 行を削除
- 欠落している `agents/openai.yaml` を生成（既存テンプレートに基づく）

50 以上の skill が触られているが、内容ではなく metadata 整合のみ。`tests/codex-skills.test.js` の validation が clean に通るようになった。

### 5.3 Unicode markers cleanup (`6c8a6bd7`, `530088c7`)

`docs/HERMES-SETUP-HOOK-ADDENDUM.md` には、初期 draft 段階の "safety violation" marker（特定の unicode 文字で囲んだ note）が残っていた。公開向け文書としては review 痕跡を含めずクリーンに出すべきという判断で、unicode marker を全て除去した。

これは review process が改善された傍証でもある。最初は draft 上で safety review を marker で残し、release surface 集約の段階で除去する、という workflow が成立している。

### 5.4 投稿された調査レポートの集約 (`d2ee1323`)

`d2ee1323` は本リポジトリの `.history/` に 3 本の調査レポートを追加するコミットである。

- `2026-04-21_plugin-install-surface-hardening/INVESTIGATION.md`
- `2026-04-22_continuous-learning-v2-entrypoint-fixes/INVESTIGATION.md`
- `2026-04-22_hook-runtime-windows-reliability/INVESTIGATION.md` + RUNBOOK.md

これらは RC1 リリース直前に行われた audit、CLv2 fix、hook runtime fix の調査記録で、後の operator 解析の materials として残された。RC1 release surface に「直近の修正履歴がドキュメント化されている」ことを含める意図がある。

### 5.5 CI catalog validator (`b6b5b6d0`)

ECC は agent 数、skill 数、command 数を README に明示している。これらが手で更新されると drift しやすいため、`scripts/validate-catalog.js` が CI で実行されている。`b6b5b6d0` はこの validator にエッジケース（empty directory、symbolic link、文字化けファイル名）の test coverage を追加した。RC1 surface での catalog 数 (38 agents, 156 skills, 72 commands) の信頼性を担保している。

### 5.6 Release workflow 改行正規化 (`2eaafc38`)

`.github/workflows/release.yml` の `run:` block 内で、Windows の CRLF と Unix の LF が混在していたのを LF に統一する patch。CI 上で `yaml.safeLoad` が改行差で挙動を変えるケースを避ける。

---

## 6. テスト状況

RC1 release surface に伴って追加されたテストは次の通り。

| テスト | 行数 | 検証対象 |
|--------|------|---------|
| `tests/docs/ecc2-release-surface.test.js` | 120 行（新規） | release surface 文書の存在、link、version 整合性 |
| `tests/plugin-manifest.test.js` | +17 行 | plugin manifest の version と name |
| `tests/scripts/release-publish.test.js` | +12 行 | release.sh の output と npm publish の整合 |
| `tests/scripts/release.test.js` | 78 行（新規） | release script の各段階の validation |
| `tests/hooks/observer-memory.test.js` | +57 行 | observer の SIGUSR1 retry |

これらは「RC1 surface が今後 release engineering の手作業で壊れない」ことを保証する仕掛けである。手で更新する余地が小さくなり、ある意味で repo を release engineering の自動化の正当な材料にした。

---

## 7. 調査所見のまとめ

### 実装状態の総括

| 領域 | 状態 | 影響 |
|------|------|------|
| Release notes / launch collateral | 完成 | 高（一括 review surface） |
| Version bump 整合 | 完成 | 高（9 翻訳含む全 manifest） |
| Cross-harness portability model | 文書化完成 | 高（ECC の positioning 確定） |
| Hermes imports skill | 規約 skill 化完成 | 高（private workflow 取り込み path 開通） |
| Observer SIGUSR1 retry | 完成 | 中（continuous learning v2 reliability） |
| Codex skill metadata | 正規化完成 | 中（Codex install surface clean） |
| Unicode marker cleanup | 完成 | 低（publication hygiene） |
| CI catalog validator | 強化完成 | 中（catalog count 信頼性） |

### 注目すべき設計判断

1. **Release surface を 1 commit でまとめる:** 40 files、863 insertions の `0a87323e` は意図的に巨大化されている。version、manifest、docs、tests が同時に一貫した snapshot として shipping される
2. **Architecture 文書を `docs/architecture/` に置く:** ECC の contract 文書を README から外し、専用 directory で版管理する。文書が「コードと同じ」更新頻度を持つことを認める設計
3. **Hermes は import 規約だけを公開:** Hermes 本体は private に留め、ECC は「他者の private workflow を取り込む方法」を skill で公開する。境界線を明示
4. **release engineering を repo で進める:** launch checklist、x thread draft、telegram handoff まで repo に置く。reviewable な surface で release を進める
5. **過去の調査レポートを RC1 surface に組み込む:** `d2ee1323` で 3 本の `.history/` を追加することで、release が「最近の修正履歴を持つ snapshot」として shipping される

### 関連調査

- [DEEP-DIVE-CROSS-HARNESS.md](./DEEP-DIVE-CROSS-HARNESS.md) — cross-harness portability model の詳細
- [DEEP-DIVE-HERMES-IMPORTS.md](./DEEP-DIVE-HERMES-IMPORTS.md) — Hermes imports skill の運用詳細
- [`2026-04-13_auto-update-command/`](../2026-04-13_auto-update-command/INVESTIGATION.md) — `2006d2ee` で同期された auto-update runtime publish
- [`2026-04-21_plugin-install-surface-hardening/`](../2026-04-21_plugin-install-surface-hardening/INVESTIGATION.md) — `d2ee1323` で RC1 surface に組み込まれた past investigation
- [`2026-05-13_ecc2-deep-dive/`](../2026-05-13_ecc2-deep-dive/00-index.md) — RC1 時点での ECC 2.0 全体像を発表用にまとめた連作記事
