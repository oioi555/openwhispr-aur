## Context

This repository (`openwhispr-aur`) was scaffolded by GitHub Copilot but does not reflect the real, tested AUR packages. The actual working packages live at `~/aur/openwhispr-bin/` and `~/aur/openwhispr-vulkan/`, with the Vulkan variant having been built and verified on a Radeon RX 6650 XT.

The implementation converges this repository to match the tested package structure, plus adds CI automation for upstream version detection and AUR metadata updates. CI does not try to prove full package builds or runtime GPU behavior.

## Goals / Non-Goals

**Goals:**
- Rewrite both PKGBUILDs to match the working, tested packages
- Rewrite the CI workflow with upstream version detection, automated fork tag resolution, checksum regeneration, `.SRCINFO` regeneration, AUR push, and maintenance-repository sync
- Generate correct `.SRCINFO` files
- Update README to reflect the actual package structure

**Non-goals:**
- CI step deduplication (loop/matrix refactor — future change)
- ARM/aarch64 support
- First-time AUR publication (manual step outside this change)
- Full CI package builds, GPU runtime validation, or transcription correctness checks

## Decisions

### D1: Specs describe the verified working state, not the Copilot template

The PKGBUILDs are written based on the tested packages at `~/aur/openwhispr-vulkan/` and `~/aur/openwhispr-bin/`, not the Copilot-generated template.

### D2: `_whisper_cpp_ver` as a separate package variable

The whisper.cpp fork version is tracked as `_whisper_cpp_ver` in the vulkan PKGBUILD, separate from `pkgver`. CI resolves this value relative to the selected OpenWhispr release instead of blindly using the latest fork tag.

### D3: Shared install path `/opt/openwhispr/`

Both packages install to `/opt/openwhispr/`. This ensures clean switching between packages via pacman without leftover paths. The launcher runs `/opt/openwhispr/open-whispr --no-sandbox`.

### D4: Fork-only whisper.cpp build

The whisper.cpp binary MUST be built from `OpenWhispr/whisper.cpp` fork, not upstream `ggerganov/whisper.cpp`. Using upstream causes GGML model incompatibility.

### D5: Automated fork-tag detection via date correlation

OpenWhispr's build script (`scripts/download-whisper-cpp.js`) does not pin the whisper.cpp version — it fetches the latest release from the fork at build time. The CI resolves the correct tag by correlating release dates:
1. Get OpenWhispr release `published_at`
2. Get all fork releases with `published_at`
3. Latest fork release where `published_at ≤ OpenWhispr release date` → `_whisper_cpp_ver`

### D6: CI uses Ubuntu as the host and Arch only for package metadata operations

The check job runs directly on `ubuntu-latest` because it only needs `curl` and `jq`.

The update job runs on a GitHub-hosted `ubuntu-latest` runner with an `archlinux:latest` container. All Arch-specific operations (`pacman`, `updpkgsums`, and `makepkg --printsrcinfo`) run inside that container. This workflow updates package metadata only; it does not run a full `makepkg` package build. Inside the Arch container, package installation uses a full sync upgrade (`pacman -Syu`) to avoid partial-upgrade breakage, and maintenance-repo git writes run as the `builder` user to avoid ownership issues.

### D7: No install scripts needed

Arch Linux pacman hooks automatically run `update-desktop-database` when packages install files to `/usr/share/applications/`. Separate `.install` scripts are unnecessary.

### D8: Copy upstream license and notice files without reconstruction

The package copies license and notice files already present in the upstream Linux release artifact (for example `LICENSE*` and `LICENSES*`) into `/usr/share/licenses/${pkgname}/`. It does not synthesize, embed, or reconstruct replacement license text that is not present in the upstream artifact.

### D9: Vulkan package must fail closed

If the Vulkan-built `whisper-server` is missing, `openwhispr-vulkan` must fail packaging instead of silently shipping the bundled CPU fallback. A misleading successful build is worse than a hard failure.

## Risks / Trade-offs

| Risk | Impact | Mitigation |
|---|---|---|
| Upstream uses `WHISPER_CPP_VERSION` env override | Date correlation returns wrong fork tag | No override found in current CI; add log note in workflow |
| Fork tag and main app bump together | Both fields change simultaneously | CI handles flags independently — no conflict |
| whisper.cpp fork build flags change | `build()` breaks | Pin CMake flags in spec; review on fork release |
| CI metadata update does not run full `makepkg` builds | Build failures are not caught by the scheduled update workflow | Keep CI scope explicit; use a separate manual build-check change if needed |
