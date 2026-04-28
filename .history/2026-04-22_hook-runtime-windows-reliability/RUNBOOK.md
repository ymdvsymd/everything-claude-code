# Hook Runtime / Windows 修復ランブック

**作成日:** 2026-04-28
**対象バージョン:** everything-claude-code post-v1.10.0
**対象者:** Claude Code v2.1.116 前後、Windows、または continuous-learning hook の重複実行で問題を調査する利用者

---

## 目次

1. [前提条件](#1-前提条件)
2. [必須と任意の切り分け](#2-必須と任意の切り分け)
3. [Claude Code argv duplication の修復](#3-claude-code-argv-duplication-の修復)
4. [PowerShell 5.1 環境での注意](#4-powershell-51-環境での注意)
5. [session summary の上書き確認](#5-session-summary-の上書き確認)
6. [やらなくてよいこと](#6-やらなくてよいこと)

---

## 1. 前提条件

このランブックは、`docs/fixes/` に追加された hook 修復スクリプトを使う。対象は主に Windows の Claude Code 環境だが、session summary の上書きや stale replay guard は macOS/Linux でも関係する。

必要なものは以下である。

| 項目 | 必須 | 理由 |
|------|------|------|
| Git Bash または `bash` | Windows では必須 | hook command の第一 token として使う |
| PowerShell | Windows では必須 | `*.ps1` 修復スクリプトを実行する |
| `~/.claude/settings.local.json` | 既存環境では必須 | hook command の修復対象 |
| ECC repo checkout | 必須 | `docs/fixes/` のスクリプトを実行する |

---

## 2. 必須と任意の切り分け

| 状況 | 必須作業 | 任意作業 |
|------|----------|----------|
| Claude Code v2.1.116 で continuous-learning hook が壊れる | `patch_settings_cl_v2_simple.ps1` または `install_hook_wrapper.ps1` を使う | 古い backup の整理 |
| plugin install 後に hook が二重実行される | 重複した manual hook block を削除する | full reinstall |
| Stop hook summary が上書きされる | repo 更新で `session-end.js` 修正を取り込む | 既存 `.tmp` summary の手動整理 |
| SessionStart で古い command が再実行される | repo 更新で `session-start.js` 修正を取り込む | 古い summary の削除 |

重要: plugin install 済みの場合、`./install.sh --profile full` や `npx ecc-install --profile full` を追加で実行する必要はない。plugin が skills、commands、hooks を読み込むため、full installer を重ねると重複 runtime behavior の原因になる。

---

## 3. Claude Code argv duplication の修復

Windows で continuous-learning hook が `.sh` 直接実行や argv duplication に巻き込まれている場合、hook command を `bash ".../observe-wrapper.sh" pre` / `post` の形に直す。

まず simple patcher で settings file を作る、または既存 file に hook entry を追加する。

```powershell
powershell -NoProfile -File docs/fixes/patch_settings_cl_v2_simple.ps1
```

既存 wrapper を配置し、hook command を wrapper へ向け直す場合は次を使う。

```powershell
powershell -NoProfile -File docs/fixes/install_hook_wrapper.ps1
```

スクリプトは `settings.local.json.bak-<timestamp>` を作ってから書き換える。成功後、`~/.claude/settings.local.json` の command は概ね次の形になる。

```json
"command": "bash \"C:/Users/.../.claude/skills/continuous-learning/hooks/observe-wrapper.sh\" pre"
```

---

## 4. PowerShell 5.1 環境での注意

Windows PowerShell 5.1 では `ConvertFrom-Json -AsHashtable` が使えない。また single-element array が JSON object として出力されることがある。追加された修復スクリプトはこの問題を避けるため、`PSCustomObject` を Hashtable と `System.Collections.ArrayList` に変換してから JSON を書き戻す。

そのため、PowerShell 7 (`pwsh`) がない環境でも、次の実行形式でよい。

```powershell
powershell -NoProfile -File docs/fixes/install_hook_wrapper.ps1
```

重要: 手で `ConvertTo-Json` を実行して settings を再生成すると、`hooks` array shape が崩れることがある。修復スクリプトを使う。

---

## 5. session summary の上書き確認

Stop hook summary の上書きが疑われる場合は、repo 更新後に `scripts/hooks/session-end.js` が transcript UUID 由来 shortId を使っていることを確認する。修正後は `transcript_path` の UUID 末尾 8 文字を使うため、親 session と subprocess の filename が分かれる。

確認観点は以下である。

| 観点 | 期待値 |
|------|--------|
| `transcript_path` が UUID `.jsonl` | shortId は UUID 末尾 8 文字 |
| UUID が大文字 | filename は lowercase |
| stdin に path がない | `CLAUDE_TRANSCRIPT_PATH` fallback を使う |
| fallback もない | 従来の `getSessionIdShort()` に戻る |

この変更は既存 summary file の削除は行わない。古い `.tmp` file が残っていても、新しい Stop hook が別 filename で書くようになれば上書きリスクは下がる。

---

## 6. やらなくてよいこと

重要: `agents` や `hooks` を `.claude-plugin/plugin.json` に手で追加しない。Claude Code plugin validator はこれらを convention で扱うため、manifest に明示すると validation error や重複 loading の原因になる。

重要: plugin install 済みの環境で full manual install を重ねない。rules だけを手動 copy する導線と、full manual install 導線は別物として扱う。

重要: Windows の hook command で `.sh` file を第一 token にしない。`bash "path/to/script.sh" arg` の形にする。

