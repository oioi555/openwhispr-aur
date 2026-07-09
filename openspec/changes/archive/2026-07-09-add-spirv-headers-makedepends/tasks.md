## 1. PKGBUILD 修正

- [x] 1.1 `openwhispr-vulkan/PKGBUILD` の `makedepends` に `'spirv-headers'` を追加し、アルファベット順 (`cmake / gcc / shaderc / spirv-headers / vulkan-headers`) を維持する — *Spec: package-architecture › Build dependencies*
- [x] 1.2 `.SRCINFO` を `makepkg --printsrcinfo` で再生成し、`spirv-headers` が `makedepends` に反映されていることを確認する

## 2. 検証

- [x] 2.1 `bash -n openwhispr-vulkan/PKGBUILD` で構文エラーが無いことを確認する
- [x] 2.2 `namcap openwhispr-vulkan/PKGBUILD` で依存関係警告が出ないことを確認する (namcap が利用可能な場合)
