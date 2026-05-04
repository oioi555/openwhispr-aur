## ADDED Requirements

### Requirement: Scheduled and manual triggering

The CI workflow MUST support both automatic and manual execution.

#### Scenario: Daily scheduled run
- **WHEN** the cron triggers at 08:00 UTC daily
- **THEN** the workflow starts the version check process

#### Scenario: Manual dispatch
- **WHEN** a maintainer triggers `workflow_dispatch`
- **THEN** the same version check and update process runs

#### Scenario: Concurrency control
- **WHEN** a workflow is already running
- **THEN** a new triggered run waits (does not cancel the in-progress run)

---

### Requirement: Version check job

A `check` job MUST determine whether updates are needed.

#### Scenario: API queries with retry
- **WHEN** the check job queries GitHub Releases API
- **THEN** it retries failed requests
- **AND** it queries `OpenWhispr/openwhispr` for the selected app release
- **AND** it queries `OpenWhispr/whisper.cpp` releases to resolve the compatible fork tag by date correlation

#### Scenario: Parsed versions are validated
- **WHEN** upstream version strings are extracted from API responses
- **THEN** the workflow validates the expected semantic version format
- **AND** it emits a clear error message before exiting if parsing fails

#### Scenario: Version extraction
- **WHEN** API responses are received
- **THEN** the OpenWhispr version has `v` prefix stripped
- **AND** the whisper.cpp fork version is resolved via date correlation: the latest `OpenWhispr/whisper.cpp` release where `published_at ≤ OpenWhispr release date`
- **AND** both are compared against current PKGBUILD values

#### Scenario: Output flags
- **WHEN** version comparison completes
- **THEN** the job outputs: `upstream_app_version`, `upstream_whisper_version`, `current_app_version`, `current_whisper_version`, `needs_update_app`, `needs_update_whisper`, `needs_update`

---

### Requirement: Package update job

An `update` job MUST run when any update is needed.

#### Scenario: Container choice
- **WHEN** the update job runs
- **THEN** it runs on a GitHub-hosted `ubuntu-latest` runner with an `archlinux:latest` container
- **AND** all Arch-specific package metadata operations run inside that container
- **AND** package installation inside the container uses a full sync upgrade rather than `pacman -Sy` partial-upgrade behavior

#### Scenario: Metadata update scope
- **WHEN** an upstream version change is detected
- **THEN** the workflow updates PKGBUILD version fields
- **AND** recalculates source checksums with `updpkgsums`
- **AND** regenerates `.SRCINFO` with `makepkg --printsrcinfo`
- **AND** pushes updated package metadata to AUR and the maintenance repository

#### Scenario: No full build validation
- **WHEN** the workflow runs
- **THEN** it does not perform a full `makepkg` package build
- **AND** it does not validate GPU runtime behavior
- **AND** it does not validate transcription correctness

#### Scenario: Job timeout
- **WHEN** the check or update job runs
- **THEN** each job sets a finite timeout to avoid hanging indefinitely

#### Scenario: Builder user setup
- **WHEN** the container starts
- **THEN** a non-root `builder` user is created with sudo access
- **AND** AUR SSH key is configured for `aur.archlinux.org` access

#### Scenario: openwhispr-bin update (when app version changed)
- **WHEN** `needs_update_app` is true
- **THEN** `pkgver` and `pkgrel=1` are set in `openwhispr-bin/PKGBUILD`
- **AND** `updpkgsums` recalculates the source tarball sha256
- **AND** `.SRCINFO` is regenerated via `makepkg --printsrcinfo`
- **AND** changes are committed and pushed to the AUR repository
- **AND** changes are committed to the maintenance repository

#### Scenario: openwhispr-vulkan update (when app or whisper.cpp changed)
- **WHEN** `needs_update_app` or `needs_update_whisper` is true
- **THEN** the appropriate version variable(s) are updated in `openwhispr-vulkan/PKGBUILD`
- **AND** `updpkgsums` recalculates sha256sums for changed sources
- **AND** `.SRCINFO` is regenerated
- **AND** changes are committed and pushed to the AUR repository
- **AND** changes are committed to the maintenance repository

#### Scenario: AUR SSH authentication
- **WHEN** pushing to AUR
- **THEN** the workflow uses the `AUR_SSH_PRIVATE_KEY` secret (ed25519 key registered on aur.archlinux.org)
- **AND** the AUR host key is pinned: `aur.archlinux.org ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEuBKrPzbawxA/k2g6NcyV5jmqwJ2s+zpgZGZ7tpLIcN`

#### Scenario: Maintenance repo sync
- **WHEN** any package is updated
- **THEN** the updated PKGBUILD and .SRCINFO are also committed and pushed to the maintenance repository (`oioi555/openwhispr-aur`)
- **AND** maintenance-repository git writes run as the unprivileged `builder` user so the checkout ownership stays consistent

---

### Requirement: Step summary reporting

Each workflow run MUST produce a GitHub Actions step summary.

#### Scenario: Summary content
- **WHEN** the workflow completes
- **THEN** the summary shows which versions were detected, whether updates were needed, and the result (updated / skipped / error)
- **AND** package-specific summary lines distinguish between an actual update and an already-up-to-date no-op

#### Scenario: Error reporting
- **WHEN** an API query fails after all retries
- **THEN** the workflow fails with a clear error message in the step summary
