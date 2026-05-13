# Install Surface RC1 Follow-ups と Runtime Resilience 調査レポート

**調査日:** 2026-05-13
**対象バージョン:** everything-claude-code post-v2.0.0-rc.1（2026-04-29〜2026-04-30 の 15 件超の修正群）
**調査者:** Claude Opus 4.7
**対象領域:** Cursor namespace、no-hooks install profile、session start skill injection、Claude plugin bundled MCPs、Cursor nested AGENTS pollution、canonical Anthropic skill duplicates、legacy command shim retirement、opencode shell probes、gateguard、plan command、block-no-verify、continuous learning observe routing、InsAIts Windows shims

---

## 1. post-v2.0.0-rc.1 で何が変わったのか

v2.0.0-rc.1 をリリースした直後の 24〜36 時間で、ECC は 2 つの並行ワークを進めた。一つは [Test Coverage Push](../2026-04-29_test-coverage-push/INVESTIGATION.md) で扱った coverage 投資。もう一つは本レポートが扱う、**ユーザー側の install surface に残っていた具体的な不快感を一気に解消する polish 作業群** である。

polish の対象は大きく 2 軸に分かれる。

第一軸は **install surface の polish** で、Cursor 向けに agent 名前空間を導入し、no-hooks minimal install profile を作り、Claude plugin が bundled MCP を強制 install するのをやめ、Cursor 配下に root `AGENTS.md` を持ち込まないようにし、Anthropic の canonical skill と被る ECC 同梱 skill を整理し、legacy command shim を default surface から外した。

第二軸は **runtime resilience の硬化** で、gateguard の concurrent state writes、destructive git detection の判定、plan command の planner-agent 不在時 fallback、block-no-verify の shell word parsing、continuous learning v2 の observe hook routing、InsAIts monitor の Windows Python shim 対応、pre-bash linter の Windows wrapper 対応など。

ここでの **install surface** は、ユーザーが ECC を install したときに各 harness ディレクトリに展開される asset の集合を指す。`~/.claude/`、`~/.cursor/`、`.codex/`、`.opencode/` 配下のすべてを含む。また、ここでの **runtime resilience** は、install 後に hook や script が実行されるときの「想定外入力に対する graceful な fallback」を指し、新機能ではなく既存挙動の堅牢化を意味する。

---

## 2. Install surface の polish

### 2.1 Cursor agent の名前空間化 (`e1d6d853`)

Cursor IDE の install では、ECC が提供する agent 群を `.cursor/agents/` 配下に置く。RC1 までは file 名が `<agent>.md`（例: `planner.md`、`code-reviewer.md`）で、ユーザーが別 marketplace から install した agent と collision するリスクがあった。

修正は、ECC が出す agent を `.cursor/agents/ecc-<agent>.md` に prefix する。`scripts/lib/cursor-agent-names.js`（26 行）が prefix の正規化を担い、`install-executor.js` と `cursor-project.js` が install path 計算に組み込む。

```js
function toCursorAgentFileName(fileName) {
  if (!fileName || fileName.startsWith('ecc-')) {
    return fileName;
  }
  return `ecc-${fileName}`;
}
```

これにより、ECC の planner と他 marketplace の planner が同居しても、それぞれ `ecc-planner.md` と `planner.md` として独立する。README の Cursor section も更新され、`ecc-*` prefix が install されることが明示された。

加えて、Cursor support の publicな表現が「**full** Cursor IDE support」から「Cursor IDE support with hooks, rules, agents, skills, commands, and MCP configs adapted for Cursor's project layout」に変更された。Claude Code との完全 parity を主張しない、honest な position に整えている。

### 2.2 no-hooks minimal install profile (`5881554a`)

ある種のユーザー（CI 環境、ペアプロ用一時環境、conservative な評価者）は hook を入れずに ECC を試したい。RC1 までは hook の install を skip する分岐がなく、`/install` を実行すると必ず hook が `~/.claude/settings.json` にマージされていた。

修正は `manifests/install-profiles.json` に `no-hooks` profile を追加し、`install-apply.js` が profile に基づいて hook block を skip するように変更した。

```json
{
  "profiles": {
    "no-hooks": {
      "description": "Install ECC skills, rules, commands, and MCP without hooks.",
      "include": ["skills", "rules", "commands", "mcp"],
      "exclude": ["hooks"]
    }
  }
}
```

README にも「hook を入れたくないなら `/install no-hooks`」というガイドが追加された。テスト 35 行ぶん（`install-manifests.test.js` と `install-apply.test.js`）でこの profile が正しく resolve されることを担保している。

### 2.3 Session start で learned skill を inject (`d26d66fd`)

これは install surface 直接ではなく session 起動時の behavior 変更だが、関連が深い。continuous-learning v2 が session 終了時に「学んだ skill」を `~/.claude/learned-skills/*.md` のような形で保存している。次の session 起動時、これらを `system_prompt` の追加文脈として注入する仕組みがなかった。

`scripts/hooks/session-start.js` に 122 行ぶんの追加が入り、`~/.claude/learned-skills/` を scan して、relevance あるものを session_start 時に inject する。`tests/hooks/hooks.test.js`（+59 行）で injection の挙動が検証されている。

これにより、ECC が「session 単位の学習を次に活かす」path が一段確立される。記事 02 の Context Graph と orthogonal な、もう一つの memory layer である（learned-skills は file-based、Context Graph は SQLite-based）。

### 2.4 Claude plugin の bundled MCPs を disable (`0c61710c`)

Claude Code plugin として ECC を install すると、ECC が同梱している MCP server config（`.mcp.json`）が plugin 経由で自動 activate される動作があった。これが「ECC を入れただけで意図しない MCP が動く」という UX 問題を起こしていた。

修正は `.claude-plugin/plugin.json` から bundled MCP の宣言を削除し、ECC の MCP は `mcp-configs/` 配下に依然存在するが、user が明示的に `/install-mcp` または `~/.claude/.mcp.json` に追加するときだけ動くようにした。

これは「ECC は asset を提供するが、勝手に有効化しない」という原則を強化する変更で、特に security-conscious な user の懸念を解消する。

### 2.5 Cursor nested AGENTS context pollution の回避 (`d49f0329`)

Cursor は nested `AGENTS.md` をディレクトリ context として読む仕様がある。これは Cursor の helpful な機能だが、ECC が host project 配下に `.cursor/AGENTS.md` を install すると、その AGENTS.md が ECC repository 自身の identity を持ち込んでしまい、host project の prompt 文脈を汚染する。

修正は、ECC の Cursor install から root `AGENTS.md` のコピーを除外することである。`.cursor/agents/ecc-*.md` という explicit な agent definition は残り、Cursor build が project agent を expose する限り、これらは reference として読まれる。一方、project の identity を ECC repo に置き換えるような副作用は出ない。

README の Cursor section に "Cursor Loading Notes" subsection が追加され、この設計判断が明文化された。

### 2.6 Anthropic canonical skill との重複整理 (`95ce9eaa`)

ECC は `.agents/skills/claude-api/SKILL.md` と `.agents/skills/frontend-design/SKILL.md` を含む数 skill を持っていた。だが、これらは Anthropic 公式のカノニカル skill と内容が大幅に重複しており、ECC で別途 maintain することの意味が薄かった。さらに、内容が乖離した場合、ユーザーがどちらを「正」と認識すべきか曖昧になる。

修正は、Anthropic canonical と重複する skill を ECC repo から削除する。`claude-api/`、`frontend-design/`、その他数件が対象。合わせて関連する `agents/openai.yaml` も削除されている。

これは ECC が「Anthropic canonical を尊重し、ECC 自身は補完的な領域に集中する」という positioning を strengthen する。重複を抱えないことで、ECC の中身が「他で得られないもの」に絞られる。

### 2.7 Legacy command shim の default surface からの退役 (`06f9eca8`)

ECC は古い command 名（例: `/old-feature-name`）から新しい command への redirect を shim として持っていた。これらは ECC v1.x 時代のユーザー向け compatibility layer として導入されたが、v2.0 で OPS が新しい canonical 名に十分慣れた判断から、default install surface から外された。

shim 自体は repo に残っているが、`manifests/install-profiles.json` の default profile が shim を include しなくなった。`/install legacy-shims` で明示的に opt-in しないと install されない。

これにより、新規ユーザーは canonical な command 名だけが見える状態で ECC に触れる。

### 2.8 OpenCode の shell file probe を避ける (`affbd334`)

OpenCode plugin 側で、shell hook の存在を検出するため `fs.existsSync('/bin/sh')` のような probe を走らせる箇所があった。これが container 環境や custom shell で false negative を起こし、hook 自体が無効化される問題を引き起こしていた。

修正は probe を削除し、OpenCode が直接 shell command を実行できる前提に切り替える。失敗時は graceful に skip。container や non-POSIX 環境でも ECC の OpenCode adapter が動くようになった。

---

## 3. Runtime resilience の硬化

### 3.1 Gateguard concurrent state writes の保護 (`c3ea7a1e`)

Gateguard hook が複数 session から同時に呼ばれると、state file（`~/.claude/cache/gateguard/state.json` 等）への concurrent write で race condition が発生していた。「読む → 変更する → 書く」の間に別 session が同じ state を別の意図で更新すると、後勝ちで上書きされる。

修正は atomic な rename pattern を強化し、`fs.writeFile` の前に file lock を取る形に変更した。`flock`-like な機構を Node.js 上で実装することで、序列性を保証している。

### 3.2 Gateguard destructive git detection の精緻化 (`1188aeaf`)

ECC の gateguard は、`git reset --hard` や `git push --force` のような destructive command を検出して block するが、`destructive` の判定が雑だった。例えば、`git reset --soft HEAD~1` は destructive ではないのに block されたり、`git push --force-with-lease` を `--force` と誤判定する case があった。

修正は git subcommand と flag を **shell word 単位で正しく parse** し、判定 logic を sub-flag に基づいて精緻化する。これは関連の `0dcde133` (block-no-verify shell word parsing) と同じ系列の修正である。

### 3.3 Plan command の planner-agent 不在時 fallback (`17aafc45`)

`/plan` command は内部で planner agent を呼び出す設計だったが、planner agent が install されていない環境（`no-hooks` profile や agent を skip した install）で `/plan` が起動できないというユーザビリティ問題があった。

修正は、planner agent が見つからない場合に inline で plan を組み立てる minimal fallback を提供する。planner agent が存在する場合はそれを使い、不在ならば command 内で完結する。これにより `/plan` は ECC profile に関わらず動く safety net を持つ。

### 3.4 block-no-verify の shell word parsing (`0dcde133`)

`block-no-verify` hook は `git commit --no-verify` のような hook bypass flag を detect する。RC1 までは raw command string に `--no-verify` が含まれるかを `String.includes()` で判定していた。これは `--no-verify-pgp` や `--no-verify-host` のような無関係なフラグも誤検出していた。

修正は `shlex` のような shell word parser で command を tokenize し、tokenize 後の word が完全一致するかで判定する。`includes` の false positive がなくなり、また `--no-verify=true` のような変形も正しく扱う。

### 3.5 Continuous learning v2 の observe hook routing (`3fadc378`)

continuous-learning v2 の observe hook は元々 `observe.sh` を直接 entry point として使っていた。Windows 環境では `.sh` 直接実行が EFTYPE で落ちる、locale や path separator の問題がある、といった移植性問題があった。

修正は、observe hook の hook entry を `node scripts/hooks/cl-observe-bridge.js` に変更し、bridge script が内部で Node.js 経由で observe ロジックを呼ぶ。Windows、macOS、Linux 共通の path を取れる。

```diff
-      "command": "${CLAUDE_PLUGIN_ROOT}/skills/continuous-learning-v2/hooks/observe.sh"
+      "command": "node ${CLAUDE_PLUGIN_ROOT}/scripts/hooks/cl-observe-bridge.js"
```

### 3.6 InsAIts monitor の Windows Python shim 対応 (`63485a26`, `1c2d5dd3`, `fe40a3d2`)

InsAIts monitor は Python subprocess を生成して LLM API を叩く監視 hook である。Windows 環境では、Python が `python.exe` ではなく App Installer の Redirector stub（`AppInstallerPythonRedirector.exe`）で resolved されるケースがあり、subprocess が hang する問題があった。

`63485a26` は Windows での Python 解決を refine する。Path lookup の優先順位を「Cygwin Python → Python launcher → conda Python → App Installer Python」に変え、Redirector stub に当たらないようにする。
`1c2d5dd3` は monitor errors で fail open する。Python が見つからない、subprocess が timeout する、API key が空のいずれでも、ECC 全体を block しないで graceful に skip。
`fe40a3d2` は hook bootstrap と InsAIts monitor の test coverage を 140 行ぶん追加（Test Coverage Push の一部）。

### 3.7 Pre-bash linter の Windows wrapper 対応 (`b5bdd935`, `ebf0d432`)

pre-bash hook の commit-quality linter は、`shellcheck` のような external tool を呼び出す。Windows では PATH に `shellcheck.exe` が無いケースが多く、毎回 hook が失敗していた。

修正は wrapper script (`scripts/hooks/win-shim/shellcheck.cmd`) を導入し、PATH に shim を載せて external tool 不在時に graceful に skip する。`ebf0d432` は test 側で Windows path 期待値の正規化を行い、`b5bdd935` は実装側で wrapper を読みに行く logic を追加。

---

## 4. なぜ post-RC1 にこれだけ集中したのか

RC1 release surface が落ち着いた直後の 24-36 時間で、これだけ多種の修正が並行して走るのは、release 直後特有の現象である。

第一に、RC1 release notes と CHANGELOG が公開された瞬間に、user feedback が一気に集まる。多くは「自分の環境では動かない」「想定外の install が起きた」という不快感系で、これらを矢継ぎ早に対処する必要がある。

第二に、release engineering が release 直前は「機能追加凍結」になるため、ある種の不快感修正が積み上がっていた。RC1 が出た瞬間に、それらの修正をまとめて push できる窓が開く。

第三に、coverage push と並行で進めることで、修正と test 追加が同じ branch で動く。新規修正がすぐ test cover される、というポジティブなサイクルが生まれる。

選ばれなかった代替案として、これらの修正を v2.0 GA まで貯めて一括で出す方向もあった。これは GA の release notes を充実させる一方、user は RC1 期間中に不快感を抱え続けることになる。今回の選択は「RC1 期間中も継続的に polish を流す」という方向で、user trust の build に効く。

---

## 5. テスト状況

このフォルダがカバーする修正のうち、テスト追加があるものを抜粋する（[Test Coverage Push](../2026-04-29_test-coverage-push/INVESTIGATION.md) と部分重複する）。

| 修正 | 追加 / 変更テスト |
|------|----------------|
| Cursor agent namespace | `tests/lib/cursor-agent-names.test.js`（新規） |
| No-hooks profile | `tests/lib/install-manifests.test.js` + `install-apply.test.js`（合計 +35 行） |
| Session start skill injection | `tests/hooks/hooks.test.js`（+59 行） |
| Block-no-verify shell word parsing | `tests/hooks/block-no-verify.test.js`（+40 行） |
| Plan command planner fallback | `tests/commands/plan.test.js`（+30 行） |
| Gateguard concurrent writes | `tests/hooks/gateguard-fact-force.test.js`（+30 行） |
| CL observe routing | `tests/hooks/cl-observe-bridge.test.js`（新規） |
| InsAIts windows shim | `tests/hooks/insaits-monitor.test.js`（+45 行） |
| Pre-bash linter windows wrapper | `tests/hooks/pre-bash-commit-quality.test.js`（+25 行） |

新規 / 変更を合わせて 300〜400 行ぶんの test 追加。修正と coverage は連動して進行している。

---

## 6. 調査所見のまとめ

### 実装状態の総括

| 領域 | 状態 | 影響 |
|------|------|------|
| Cursor agent namespace | 完成 | 高（marketplace 競合回避） |
| No-hooks install profile | 完成 | 中（試用ユーザー導線開通） |
| Session start skill injection | 完成 | 高（CL v2 のループ閉じる） |
| Claude plugin bundled MCPs disable | 完成 | 高（不意の MCP 起動を防止） |
| Cursor nested AGENTS 回避 | 完成 | 高（host project 汚染防止） |
| Anthropic canonical skill 重複整理 | 完成 | 中（positioning sharpening） |
| Legacy command shim 退役 | 完成 | 中（new user の view を清潔に） |
| OpenCode shell probe 削除 | 完成 | 中（container 環境対応） |
| Gateguard 改善 (concurrent writes + destructive detection) | 完成 | 高（誤判定減少） |
| Plan command fallback | 完成 | 中（profile 非依存性） |
| Block-no-verify shell word parsing | 完成 | 中（false positive 解消） |
| CL observe routing through Node | 完成 | 高（cross-platform 互換性） |
| InsAIts Windows shim | 完成 | 高（Windows InsAIts 安定化） |
| Pre-bash linter Windows wrapper | 完成 | 中（Windows linting 復活） |

### 注目すべき設計判断

1. **Cursor support の表現を honest に:** 「full support」から「project layout 用 adapter」へ。完全 parity を主張しない positioning を取る
2. **MCP は asset 提供 + 明示的 opt-in:** ECC は MCP を勝手に有効化しない原則を強化
3. **canonical なものは Anthropic に譲る:** 重複を抱え込まず、ECC は補完領域に集中する
4. **shell command の parse は word-level で行う:** `String.includes()` の安易さを捨て、tokenize ベースの判定にすることで false positive を減らす
5. **shell script は Node bridge 経由で実行:** Windows 互換性を考えると、`.sh` を直接 hook entry にしない方が安全
6. **Python subprocess は fail open:** monitor が落ちても ECC 全体は止めない。安全側の落ち方を採用

### 関連調査

- [`2026-04-28_ecc2-rc1-release-surface/`](../2026-04-28_ecc2-rc1-release-surface/INVESTIGATION.md) — RC1 release 本体
- [`2026-04-29_test-coverage-push/`](../2026-04-29_test-coverage-push/INVESTIGATION.md) — 並行する coverage 投資
- [`2026-04-22_hook-runtime-windows-reliability/`](../2026-04-22_hook-runtime-windows-reliability/INVESTIGATION.md) — 1 週間前の Windows hook 修正、本フォルダの shim 系修正の前段
- [`2026-04-22_continuous-learning-v2-entrypoint-fixes/`](../2026-04-22_continuous-learning-v2-entrypoint-fixes/INVESTIGATION.md) — CL v2 entrypoint の前段修正、本フォルダの observe routing が継続
