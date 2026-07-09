## Why

OpenWhispr 1.7.4 以降、`openwhispr-vulkan` がビルドできなくなった。ユーザー (viperpaulo) から AUR コメントで報告を受けた。原因は whisper.cpp の Vulkan バックエンドが `spirv-headers` を直接要求するようになり、従来の `shaderc` 経由の間接依存では足りなくなったため。パッケージのビルド可能性が壊れているため即時修正が必要。

## What Changes

- `openwhispr-vulkan/PKGBUILD` の `makedepends` に `'spirv-headers'` を明示的に追加し、アルファベット順を維持する。

## Non-goals

- `pkgver` の 1.7.0 → 1.7.4 バンプは行わない (別件、アップストリーム追従は CI または別 Change で対応)。
- `openwhispr-bin` 側は変更しない (Vulkan ビルドを行わないため影響なし)。
- CI ワークフローに `makepkg` ビルドテストを追加しない (ユーザー指示により簡略化を維持)。

## Capabilities

### New Capabilities

(なし)

### Modified Capabilities

- `package-architecture`: `openwhispr-vulkan` のビルド依存関係要件を更新 — `spirv-headers` を makedepends に明示的に含める。

## Impact

- **対象ファイル**: `openwhispr-vulkan/PKGBUILD` のみ。
- **依存関係**: ビルド時依存に `spirv-headers` (公式 `extra` リポジトリ) を 1 つ追加。実行時依存は変更なし。
- **ユーザー影響**: AUR ユーザーが `makepkg` でビルドできるよう復旧。`spirv-headers` は `extra` に入っているため、通常環境では既に利用可能。
- **CI**: 影響なし。次回の CI 実行で AUR 側に反映される。
