## Purpose

Define how CI monitors OpenWhispr releases and resolves the compatible OpenWhispr/whisper.cpp fork tag while keeping the main app and fork versions independently tracked.

## Requirements

### Requirement: Dual upstream monitoring

The CI MUST monitor OpenWhispr releases and resolve the compatible OpenWhispr/whisper.cpp fork tag for the selected app release.

#### Scenario: OpenWhispr main app releases
- **WHEN** the CI checks for updates
- **THEN** it queries `https://api.github.com/repos/OpenWhispr/openwhispr/releases/latest` to get the latest tag
- **AND** strips the `v` prefix to obtain the version number (e.g., tag `v1.6.10` → version `1.6.10`)

#### Scenario: whisper.cpp fork version detection via date correlation
- **WHEN** the CI checks for updates
- **THEN** it fetches the OpenWhispr release's `published_at` date from GitHub API
- **AND** it fetches all `OpenWhispr/whisper.cpp` releases with their `published_at` dates
- **AND** it selects the latest fork release where `published_at ≤ OpenWhispr release date`
- **AND** that tag value becomes the resolved `_whisper_cpp_ver`
- **AND** it does not update to fork releases newer than the selected OpenWhispr release unless a future compatibility rule is defined

---

### Requirement: Independent version lifecycle

The main app version (`pkgver`) and resolved whisper.cpp fork version (`_whisper_cpp_ver`) MUST be stored separately and updated only when their resolved values change.

#### Scenario: Main app update only
- **WHEN** OpenWhispr releases a new version but the fork tag is unchanged
- **THEN** only `pkgver` is bumped in both packages
- **AND** `_whisper_cpp_ver` stays the same in `openwhispr-vulkan`
- **AND** `openwhispr-bin` only needs `pkgver` + `sha256sums` update
- **AND** `openwhispr-vulkan` needs `pkgver` + `sha256sums` for the app tarball update, but the whisper.cpp source sha256 stays the same

#### Scenario: Resolved whisper.cpp version update only
- **WHEN** the resolved fork tag differs from the current `_whisper_cpp_ver` but the OpenWhispr app version is unchanged
- **THEN** `_whisper_cpp_ver` is bumped in `openwhispr-vulkan`
- **AND** `openwhispr-bin` is not affected (it uses the bundled CPU binary)
- **AND** `openwhispr-vulkan` sha256sums for the whisper.cpp source is recalculated

#### Scenario: Both resolved values update simultaneously
- **WHEN** both the OpenWhispr app version and the resolved fork tag differ from current PKGBUILD values
- **THEN** both `pkgver` and `_whisper_cpp_ver` are bumped in `openwhispr-vulkan`
- **AND** both sha256sums are recalculated
- **AND** `openwhispr-bin` gets only the `pkgver` bump

---

### Requirement: Version change detection

The CI MUST determine which versions changed before performing updates.

#### Scenario: Comparison logic
- **WHEN** the check job runs
- **THEN** it reads `pkgver` from `openwhispr-bin/PKGBUILD` (current main app version)
- **AND** it reads `_whisper_cpp_ver` from `openwhispr-vulkan/PKGBUILD` (current fork version)
- **AND** it compares each against the upstream values
- **AND** it outputs three boolean flags: `needs_update_app`, `needs_update_whisper`, `needs_update` (true if either is true)

#### Scenario: No changes detected
- **WHEN** both upstream versions match the current PKGBUILD values
- **THEN** the workflow exits without updating anything
- **AND** the step summary notes "No update needed"

---

### Requirement: Version coupling awareness

While the values are stored separately, the CI MUST be aware that the fork version is coupled to the selected OpenWhispr release.

#### Scenario: Main app may pin a new fork version
- **WHEN** a new OpenWhispr release bundles a whisper.cpp binary built from a newer fork tag
- **THEN** the CI automatically resolves the correct `_whisper_cpp_ver` via date correlation (see "whisper.cpp fork version detection via date correlation" scenario)
- **AND** if the resolved version differs from the current `_whisper_cpp_ver`, `needs_update_whisper` is set to true

#### Scenario: Version pinning documentation
- **WHEN** either version is updated
- **THEN** the change commit message includes both versions (e.g., `chore: bump openwhispr to 1.7.0, whisper.cpp to 0.0.7` or `chore: bump openwhispr to 1.7.0`)
