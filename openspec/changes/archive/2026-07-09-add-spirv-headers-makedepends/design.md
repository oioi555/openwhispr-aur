## Context

`openwhispr-vulkan/PKGBUILD` は whisper.cpp fork を `-DGGML_VULKAN=ON` 付きで CMake ビルドする。従来 `makedepends` は `shaderc` + `vulkan-headers` だけで、`spirv-headers` は `shaderc` の依存関係として間接的に引かれることを暗黙に前提としていた。

OpenWhispr 1.7.4 以降、whisper.cpp の Vulkan バックエンド CMake スクリプトが `spirv-headers` を直接 `find_package` するよう変化し、間接依存では解決できなくなった。結果として AUR ユーザーが `makepkg` 実行時に CMake configure 段階で失敗する。

Arch パッケージングの慣行として、ビルド時に直接参照するパッケージは `makedepends` に明示的に列挙すべきであり、他パッケージの依存関係経由で間接引きすることは推奨されない (間接依存は予告無く変わる可能性があるため)。

## Goals / Non-Goals

**Goals:**
- `openwhispr-vulkan` のビルドを最新の whisper.cpp fork で復旧する。
- Arch パッケージング慣行に従い、ビルド時に直接参照する依存を makedepends に明示する。

**Non-Goals:**
- `pkgver` のバンプ (1.7.0 → 1.7.4) は行わない。別 Change または CI の自動追従で対応。
- CI へのビルドテスト追加は行わない (ユーザー指示により簡略化維持)。

## Decisions

### D1: `spirv-headers` を makedepends に追加 (アルファベット順維持)

**選択**: 既存 `makedepends` 配列に `'spirv-headers'` を挿入し、`cmake / gcc / shaderc / spirv-headers / vulkan-headers` のアルファベット順を維持する。

**理由**:
- Arch 公式リポジトリ (`extra/spirv-headers`) に存在し、ユーザー環境で利用可能。
- 明示依存により、`shaderc` 側の依存変更に影響されず安定。
- アルファベット順は現在の PKGBUILD の既存規約に合致。

**代替案と却下理由**:
- *依存を暗黙のまま放置* → 再発リスクあり。却下。
- *`spirv-tools` も追加* → whisper.cpp の configure は `spirv-tools` を直接要求しない (`shaderc` が持つ)。過剰依存になるため却下。

## Risks / Trade-offs

- **[リスク] `spirv-headers` が `extra` から削除される** → 確率は極めて低い (Khronos 公式パッケージ)。追加メンテ不要。
- **[トレードオフ] 過剰依存の可能性** → `shaderc` 経由でも解決できる環境では重複宣言になるが、Arch の慣行 (直接参照は明示) に従うべきと判断。実害なし。
