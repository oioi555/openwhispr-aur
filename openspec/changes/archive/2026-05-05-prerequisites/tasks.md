## 1. openwhispr-bin PKGBUILD

- [x] 1.1 Rewrite `openwhispr-bin/PKGBUILD` based on `~/aur/openwhispr-bin/PKGBUILD`: correct depends, optdepends, source, `package()` with `/opt/openwhispr/` install path, `/usr/bin/openwhispr` launcher with `--no-sandbox`, `.desktop` file, icon from tarball — [req: openwhispr-bin package definition]
- [x] 1.2 Regenerate `openwhispr-bin/.SRCINFO` — placeholder with `sha256sums = SKIP`; CI regenerates via `makepkg --printsrcinfo` after `updpkgsums` — [req: openwhispr-bin package definition]

## 2. openwhispr-vulkan PKGBUILD

- [x] 2.1 Rewrite `openwhispr-vulkan/PKGBUILD` based on `~/aur/openwhispr-vulkan/PKGBUILD`: dual source (app + whisper.cpp fork), `_whisper_cpp_ver` variable, `makedepends`, `build()` with CMake Vulkan flags, `package()` with binary replacement — [req: openwhispr-vulkan package definition]
- [x] 2.2 Regenerate `openwhispr-vulkan/.SRCINFO` — placeholder with `sha256sums = SKIP`; CI regenerates via `makepkg --printsrcinfo` after `updpkgsums` — [req: openwhispr-vulkan package definition]

## 3. CI Workflow

- [x] 3.1 Rewrite check job: OpenWhispr release query + whisper.cpp fork release list with date-correlation logic for compatible `_whisper_cpp_ver`, output flags (`needs_update_app`, `needs_update_whisper`) — [req: Version check job, Dual upstream monitoring, Version coupling awareness]
- [x] 3.2 Rewrite update job: ubuntu-latest host with archlinux container, builder user, AUR SSH setup, conditional per-package metadata updates (bin: pkgver only; vulkan: pkgver +/or `_whisper_cpp_ver`), `updpkgsums`, `.SRCINFO` regen, AUR push, maintenance repo push; no full CI package build — [req: Package update job]
- [x] 3.3 Add step summary reporting with detected versions and update results — [req: Step summary reporting]

## 4. Documentation

- [x] 4.1 Update `README.md` to reflect new package structure, shared install path, dual-upstream monitoring — [req: Two mutually exclusive packages]

## 5. Verification

- [x] 5.1 Run `openspec validate prerequisites` — all artifacts pass
- [x] 5.2 Diff `openwhispr-bin/PKGBUILD` against `~/aur/openwhispr-bin/PKGBUILD` — confirm alignment
- [x] 5.3 Diff `openwhispr-vulkan/PKGBUILD` against `~/aur/openwhispr-vulkan/PKGBUILD` — confirm alignment
- [x] 5.4 Review CI workflow for completeness against ci-workflow spec
- [x] 5.5 Confirm no .install files remain in repo or CI workflow

## 6. Post-review hardening

- [x] 6.1 Simplify packaged license handling so packages copy upstream artifact `LICENSE*` / `LICENSES*` notice files without synthesizing replacement license text
- [x] 6.2 Make `openwhispr-vulkan` fail the package build if the Vulkan replacement binary is missing
- [x] 6.3 Harden the GitHub Actions update job against partial upgrades (`pacman -Syu`) and maintenance-repo ownership issues (`builder` writes)
- [x] 6.4 Clarify CI scope: metadata update automation only, not full package build, GPU runtime validation, or transcription validation
- [x] 6.5 Remove direct latest-fork update policy and keep `_whisper_cpp_ver` resolved relative to the selected OpenWhispr release
- [x] 6.6 Audit runtime and build dependencies against upstream deb/rpm/electron-builder defaults and local ELF/CMake outputs: keep a minimal Electron runtime set, keep `libx11` for XWayland wrapper behavior, avoid system `ffmpeg` because `ffmpeg-static` is bundled, and add `shaderc` for Vulkan shader compilation
