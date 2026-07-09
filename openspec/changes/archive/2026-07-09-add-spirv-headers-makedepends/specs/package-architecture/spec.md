## MODIFIED Requirements

### Requirement: openwhispr-vulkan package definition

`openwhispr-vulkan` MUST build a Vulkan-enabled whisper-server from the forked source and replace the bundled CPU binary.

#### Scenario: Dual source — app + whisper.cpp fork
- **WHEN** the package is built
- **THEN** sources include:
  1. Official OpenWhispr release tar.gz (same as `openwhispr-bin`)
  2. `whisper.cpp-${_whisper_cpp_ver}.tar.gz` from `https://github.com/OpenWhispr/whisper.cpp/archive/refs/tags/${_whisper_cpp_ver}.tar.gz`

#### Scenario: Vulkan whisper-server build
- **WHEN** `makepkg` runs `build()`
- **THEN** whisper.cpp is compiled with CMake flags:
  - `-DGGML_VULKAN=ON`
  - `-DBUILD_SHARED_LIBS=OFF` (static linking)
  - `-DWHISPER_BUILD_SERVER=ON`
  - `-DWHISPER_BUILD_EXAMPLES=ON`
  - `-DWHISPER_BUILD_TESTS=OFF`
  - Target: `whisper-server`

#### Scenario: Build dependencies
- **WHEN** the package is built
- **THEN** `makedepends` includes: `cmake`, `gcc`, `shaderc`, `spirv-headers`, `vulkan-headers`
- **AND** `spirv-headers` is declared explicitly because the Vulkan backend's CMake configure step resolves SPIR-V headers directly (not transitively via `shaderc`)

#### Scenario: Vulkan is a hard dependency
- **WHEN** the package is installed
- **THEN** `vulkan-icd-loader` and `vulkan-driver` are required dependencies (not optional)

#### Scenario: Desktop runtime dependencies
- **WHEN** either package is installed
- **THEN** desktop runtime dependencies are aligned with upstream Electron Linux package defaults where practical
- **AND** `libx11` remains a hard dependency because the upstream Linux wrapper forces XWayland under Wayland sessions
- **AND** system `ffmpeg` is not a hard dependency because the upstream artifact bundles `ffmpeg-static` and only falls back to a system binary if the bundled one is unavailable

#### Scenario: Binary replacement in package()
- **WHEN** `makepkg` runs `package()`
- **THEN** the Vulkan-built `whisper-server` replaces `/opt/openwhispr/resources/bin/whisper-server-linux-x64`
- **AND** if the Vulkan binary is not found, packaging fails instead of silently keeping the CPU binary

#### Scenario: Same install path as bin
- **WHEN** the package is installed
- **THEN** the app is at `/opt/openwhispr/` (same path as openwhispr-bin), ensuring clean switching between packages

#### Scenario: Desktop integration
- **WHEN** either package is installed
- **THEN** a `.desktop` file is created at `/usr/share/applications/openwhispr.desktop`
- **AND** Arch Linux's pacman hook automatically runs `update-desktop-database` (no install script needed)

#### Scenario: Icon installation

- **WHEN** either package is installed
- **THEN** the icon is installed to `/usr/share/pixmaps/openwhispr.png`

#### Scenario: License installation

- **WHEN** either package is installed
- **THEN** license and notice files included in the upstream release tarball, such as `LICENSE*` and `LICENSES*`, are installed under `/usr/share/licenses/$pkgname/`
- **AND** the package does not synthesize or embed replacement license text that is not present in the upstream artifact
