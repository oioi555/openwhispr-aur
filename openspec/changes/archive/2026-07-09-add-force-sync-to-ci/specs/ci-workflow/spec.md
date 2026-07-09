## MODIFIED Requirements

### Requirement: Scheduled and manual triggering

The CI workflow MUST support both automatic and manual execution.

#### Scenario: Daily scheduled run
- **WHEN** the cron triggers at 08:00 UTC daily
- **THEN** the workflow starts the version check process
- **AND** `inputs.force_sync` is null (treated as false)

#### Scenario: Manual dispatch without force
- **WHEN** a maintainer triggers `workflow_dispatch` without setting `force_sync`
- **THEN** the same version check and update process runs as the scheduled run
- **AND** the `update` job runs only when `needs_update == 'true'`

#### Scenario: Manual dispatch with force sync
- **WHEN** a maintainer triggers `workflow_dispatch` with `force_sync` set to `true`
- **THEN** the `check` job still runs (to report upstream version status in the step summary)
- **AND** the `update` job runs unconditionally, regardless of `needs_update` value
- **AND** version bump `sed` steps are skipped because `NEEDS_APP` and `NEEDS_WHISPER` remain false
- **AND** `updpkgsums`, `.SRCINFO` regeneration, AUR clone/copy/commit/push, and maintenance-repo commit all execute
- **AND** the step summary shows a `🔄 Force sync triggered` marker so the run is distinguishable from normal version-bump runs

#### Scenario: Concurrency control
- **WHEN** a workflow is already running
- **THEN** a new triggered run waits (does not cancel the in-progress run)
- **AND** this applies to both scheduled and force-sync runs (single `concurrency.group: aur-update`)

---

### Requirement: Package update job

An `update` job MUST run when any update is needed, OR when force sync is requested.

#### Scenario: Container choice
- **WHEN** the update job runs
- **THEN** it runs on a GitHub-hosted `ubuntu-latest` runner with an `archlinux:latest` container
- **AND** all Arch-specific package metadata operations run inside that container
- **AND** package installation inside the container uses a full sync upgrade rather than `pacman -Sy` partial-upgrade behavior

#### Scenario: Metadata update scope
- **WHEN** an upstream version change is detected OR force sync is requested
- **THEN** the workflow updates PKGBUILD version fields (only when version actually changed)
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

#### Scenario: openwhispr-bin sync (when force sync requested but version unchanged)
- **WHEN** `force_sync` is true AND `needs_update_app` is false
- **THEN** the `pkgver`/`pkgrel` sed step is skipped
- **AND** `updpkgsums` and `.SRCINFO` regeneration still run (idempotent)
- **AND** the maintenance-repo PKGBUILD and .SRCINFO are copied to the AUR clone and pushed if a diff exists

#### Scenario: openwhispr-vulkan update (when app or whisper.cpp changed)
- **WHEN** `needs_update_app` or `needs_update_whisper` is true
- **THEN** the appropriate version variable(s) are updated in `openwhispr-vulkan/PKGBUILD`
- **AND** `updpkgsums` recalculates sha256sums for changed sources
- **AND** `.SRCINFO` is regenerated
- **AND** changes are committed and pushed to the AUR repository
- **AND** changes are committed to the maintenance repository

#### Scenario: openwhispr-vulkan sync (when force sync requested but versions unchanged)
- **WHEN** `force_sync` is true AND both `needs_update_app` and `needs_update_whisper` are false
- **THEN** the version-variable sed steps are skipped
- **AND** `updpkgsums` and `.SRCINFO` regeneration still run (idempotent)
- **AND** the maintenance-repo PKGBUILD and .SRCINFO are copied to the AUR clone and pushed if a diff exists

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
- **AND** when `force_sync` is true, the summary displays a `🔄 Force sync triggered` marker at the top

#### Scenario: Error reporting
- **WHEN** an API query fails after all retries
- **THEN** the workflow fails with a clear error message in the step summary
