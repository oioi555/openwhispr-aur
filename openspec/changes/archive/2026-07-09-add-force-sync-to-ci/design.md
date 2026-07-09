## Context

CI ワークフロー (`.github/workflows/update-aur.yml`) は2ジョブ構成:

1. **check ジョブ**: GitHub Releases API を叩いて upstream バージョンを取得し、PKGBUILD の `pkgver` / `_whisper_cpp_ver` と比較。結果を `needs_update` / `needs_update_app` / `needs_update_whisper` として出力。
2. **update ジョブ**: `if: needs.check.outputs.needs_update == 'true'` で発火。sed でバージョン番号を置換 → `updpkgsums` → `makepkg --printsrcinfo` → AUR に clone & cp & commit & push。

問題: バージョン番号が変わらない「構造変更」(依存追加、パッチ、メタデータ修正等) は check ジョブが `needs_update=false` を出すため、update ジョブが発火しない。結果、AUR 側に反映されない。

具体例: 今回 spirv-headers を makedepends に追加したが、AUR 側には反映されない。maintenance repo (GitHub) には push 済みでも、AUR 側の PKGBUILD は古いまま。

## Goals / Non-Goals

**Goals:**
- メンテナが任意のタイミングで「maintenance repo の最新 PKGBUILD/.SRCINFO を AUR に強制同期」できる手段を提供する。
- 既存の毎日スケジュール実行の挙動は一切変えない (バージョン同じなら skip のまま)。

**Non-Goals:**
- check ジョブでの PKGBUILD 内容 diff 検知は導入しない (check ジョブに AUR ssh clone を追加する複雑度に対し、手動 force_sync で十分と判断)。
- force_sync 実行時のバージョン bump は行わない (sed 置換は既に NEEDS_APP/NEEDS_WHISPER でガードされており、force_sync の時はこれらが false になるため自然にスキップ)。

## Decisions

### D1: `workflow_dispatch` に `force_sync` input を追加 (default false)

**選択**: `on.workflow_dispatch.inputs.force_sync` を boolean, default false で定義。

**理由**:
- 既存のスケジュール実行は inputs が null になるため、`false` 相当で従来通り動作。
- メンテナが手動で `force_sync=true` を指定した時だけ強制同期が走る。
- GitHub Actions の inputs は `workflow_dispatch` にしか存在しないため、スケジュール実行に影響しないことが自明。

### D2: update ジョブの `if` を `needs_update == 'true' || inputs.force_sync == true` に拡張

**選択**:
```yaml
update:
  needs: check
  if: needs.check.outputs.needs_update == 'true' || inputs.force_sync == true
```

**理由**:
- 既存の自動発火パスは完全維持。
- force_sync=true の時は check ジョブの結果に関わらず update を実行。check ジョブ自体は並走して実行され、step summary にバージョン状況は表示される (情報価値)。
- GitHub Actions では job の if 式で `inputs.*` を直接参照できる。

**代替案と却下理由**:
- *check ジョブの if で force_sync を見て needs_update を強制 true にする* → check ジョブの責務(情報提供)が汚れる。却下。
- *新しく `force-sync` ワークフローを分ける* → 重複コードが増える。却下。

### D3: package-specific update steps も `force_sync` で実行する

**選択**: `Update openwhispr-bin` と `Update openwhispr-vulkan` の step-level `if` も `inputs.force_sync == true` で実行されるよう拡張する。

**理由**:
- update job だけを force_sync で実行しても、package-specific step が旧条件 (`needs_update_*`) のままだと AUR clone/copy/push が skip され、force_sync が no-op になる。
- force_sync の目的は「maintenance repo の PKGBUILD/.SRCINFO を AUR に同期すること」なので、package-specific step 全体を実行する必要がある。
- バージョンが変わっていない force_sync では、既存値への sed / `updpkgsums` / `.SRCINFO` 再生成は idempotent で、AUR 側に差分がなければ `git diff --cached --quiet` により push は skip される。
- `openwhispr-vulkan` の `_whisper_cpp_ver` 変更は既存の `NEEDS_WHISPER` ガードを維持するため、force_sync で不意にバージョンを書き換えることはない。

### D4: ステップサマリーで force_sync 実行を明示

**選択**: force_sync 実行時に step summary で `🔄 Force sync triggered` を表示。

**理由**: 実行履歴で「なぜバージョン変わってないのに update が走ったか」を明示するため。トレーサビリティ向上。

## Risks / Trade-offs

- **[リスク] force_sync 実行時にメンテ repo と AUR の間で不整合が起きる** → update ジョブは maintenance repo の PKGBUILD/.SRCINFO を AUR に cp する設計なので、必ず maintenance repo が正となる。メンテナが先に maintenance repo を正しい状態に push しておく運用を守れば整合性は保たれる。
- **[トレードオフ] check ジョブが毎回必ず走る** → force_sync=true の時は check の結果を使わないが、API レート制限 (GitHub Releases API) への影響は軽微 (1日1回 + たまの手動実行)。許容範囲。
- **[リスク] inputs.force_sync の型解釈** → GitHub Actions では inputs は文字列として渡されることがある。式 `inputs.force_sync == true` は boolean 比較だが、文字列 `"true"` と boolean `true` が混在するケースを `== true` で安全に扱えるか注意。GitHub 公式ドキュメントでは boolean input は `true`/`false` リテラルとして式評価されるため、`inputs.force_sync == true` で正しく動作する。 YAML 検証で確認する。
