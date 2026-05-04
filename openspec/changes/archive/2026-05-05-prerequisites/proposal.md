## Why

The Copilot-generated template does not reflect the real working packages. The actual `openwhispr-vulkan` at `~/aur/openwhispr-vulkan/` builds whisper.cpp from a forked source with Vulkan support and replaces the bundled CPU binary — the template is just a copy of `openwhispr-bin` with Vulkan in hard deps. The CI workflow also only monitors a single upstream, unaware that the whisper.cpp fork must be tracked independently.

This change establishes the baseline AUR package definitions and metadata-update automation needed to converge the repository with the tested package state.

## What Changes

1. **Rewrite `openwhispr-bin/PKGBUILD`** to match the working package at `~/aur/openwhispr-bin/`: correct dependencies, install path `/opt/openwhispr/`, launcher with `--no-sandbox`, desktop file, icon.
2. **Rewrite `openwhispr-vulkan/PKGBUILD`** to match the working package at `~/aur/openwhispr-vulkan/`: dual source (app + whisper.cpp fork), `build()` with CMake Vulkan flags, binary replacement in `package()`, and hard failure if the Vulkan replacement binary is missing.
3. **Rewrite `.github/workflows/update-aur.yml`** to implement version detection and metadata updates: resolve the compatible fork tag by date correlation, update PKGBUILD version fields, regenerate checksums and `.SRCINFO`, then push package metadata to AUR and this maintenance repository.
4. **Regenerate `.SRCINFO`** files for both packages.
5. **Simplify license installation** so packages copy license/notice files included in the upstream release artifact without synthesizing replacement license text.

## Capabilities

### New Capabilities
- `package-architecture`: Correct PKGBUILD for both packages with proper dependencies, build steps, and mutual exclusion
- `upstream-monitoring`: Version detection policy for OpenWhispr releases and compatible whisper.cpp fork tags
- `ci-workflow`: GitHub Actions workflow implementing check + metadata-update jobs, without full build or runtime validation

### Modified Capabilities
<!-- None — baseline establishment -->

## Impact

- `openwhispr-bin/PKGBUILD` — rewritten from scratch
- `openwhispr-bin/.SRCINFO` — regenerated
- `openwhispr-vulkan/PKGBUILD` — rewritten from scratch
- `openwhispr-vulkan/.SRCINFO` — regenerated
- `.github/workflows/update-aur.yml` — rewritten with version detection and package metadata update logic
- `README.md` — updated to reflect new package structure
- `openspec/changes/prerequisites/*` — refined to capture packaging hardening and license handling

## Non-goals

- Refactoring duplicated CI update steps into a loop/matrix (future improvement)
- Adding ARM/aarch64 support
- Publishing to AUR for the first time (separate manual step)
- Full CI package builds, GPU runtime validation, or transcription correctness checks
