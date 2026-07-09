## Why

`openwhispr-vulkan` に `spirv-headers` を追加したが (Change `add-spirv-headers-makedepends`)、CI はバージョンが同じだと `update` ジョブをスキップする設計 (`if: needs_update == 'true'`) ので、AUR 側に反映されない。バージョン以外の構造変更 (依存追加、パッチ、メタデータ修正等) を AUR に push する手段が必要。

毎日の自動運用はバージョンベースのまま維持し、メンテナが任意のタイミングで「maintenance repo の PKGBUILD/.SRCINFO を AUR に強制同期」できる 手動トリガ を追加する。

## What Changes

- `update-aur.yml` の `workflow_dispatch` に `force_sync` boolean input (default `false`) を追加する。
- `update` ジョブの `if` 条件を拡張し、`inputs.force_sync == 'true'` の時は check ジョブの `needs_update` 結果に関わらず実行する。
- `update` ジョブ内の既存のバージョンバンプ sed ステップ (pkgver / _whisper_cpp_ver 置換) は `NEEDS_APP` / `NEEDS_WHISPER` でガード済みのため、force_sync 実行時は自然にスキップされる (sha256 は `updpkgsums` で再計算されるが、既に正しければ変化なし)。

## Non-goals

- check ジョブの「PKGBUILD 内容 diff 検知」は導入しない (check ジョブに AUR への ssh clone を追加する複雑度に対し、force_sync の手動運用で十分カバーできると判断)。
- ローカルからの直接 AUR ssh push は扱わない (CI 経由を正とする)。
- バージョン検知ロジック自体 (`UPSTREAM_APP`, `UPSTREAM_WHISPER`) の変更はしない。

## Capabilities

### New Capabilities

(なし)

### Modified Capabilities

- `ci-workflow`: `workflow_dispatch` に `force_sync` 入力を追加し、`update` ジョブがバージョン非依存で実行できる要件を追加。

## Impact

- **対象ファイル**: `.github/workflows/update-aur.yml` のみ。
- **CI 動作**:
  - 毎日のスケジュール実行: 従来通り (バージョン同じなら update スキップ)。
  - 手動 dispatch: `force_sync=true` を指定すれば、バージョン同じでも maintenance repo → AUR へ強制同期。
- **メンテナ運用**: 依存追加・パッチ等の「バージョン変わらない構造変更」が AUR に反映できるようになる。
- **リスク**: force_sync=true を誤って実行しても、maintenance repo と AUR の内容が同一なら diff なしで skip されるため、実害はない。
