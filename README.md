# openwhispr-aur

AUR maintenance repository for [OpenWhispr](https://github.com/OpenWhispr/openwhispr) packages.

| Package | Description |
|---|---|
| [`openwhispr-bin`](./openwhispr-bin/) | Official pre-built binary. CPU-only whisper.cpp inference. |
| [`openwhispr-vulkan`](./openwhispr-vulkan/) | Same app, but with a Vulkan-built whisper.cpp that replaces the bundled CPU binary. `vulkan-icd-loader` and `vulkan-driver` are hard dependencies. Packaging now fails if the Vulkan replacement binary is missing, to avoid silently shipping the CPU fallback. |

Both packages `provides=('openwhispr')` and conflict with each other (and `openwhispr-appimage`), so only one may be installed at a time. Both install the app to `/opt/openwhispr/` for clean switching.

---

## Install

```sh
# CPU binary (no GPU required)
paru -S openwhispr-bin

# Vulkan GPU variant (AMD/Intel/NVIDIA)
paru -S openwhispr-vulkan
```

---

## How the Vulkan package works

`openwhispr-vulkan` does **not** modify the OpenWhispr source. Instead:

1. It builds `whisper-server` from the [OpenWhispr/whisper.cpp](https://github.com/OpenWhispr/whisper.cpp) fork with `-DGGML_VULKAN=ON`.
2. In `package()`, the Vulkan-built binary replaces `/opt/openwhispr/resources/bin/whisper-server-linux-x64`.
3. The fork version (`_whisper_cpp_ver`) is pinned because OpenWhispr's GGML models are paired to a specific fork tag — using upstream `ggerganov/whisper.cpp` causes model incompatibility.

---

## Automated updates (GitHub Actions)

The workflow in [`.github/workflows/update-aur.yml`](.github/workflows/update-aur.yml) runs **daily at 08:00 UTC** and checks the OpenWhispr release plus the OpenWhispr/whisper.cpp fork releases:

1. **OpenWhispr main app** — checks the latest release tag (`pkgver`).
2. **whisper.cpp fork** — resolves the correct fork tag (`_whisper_cpp_ver`) by date correlation: the latest fork release published on or before the OpenWhispr release date.

If either version has changed:
- `openwhispr-bin` is updated when the main app version changes.
- `openwhispr-vulkan` is updated when either the main app or whisper.cpp fork version changes.
- Checksums are recalculated via `updpkgsums` and `.SRCINFO` files regenerated.
- Changes are pushed to each AUR repository via SSH, then committed back to this maintenance repository.

The workflow now also performs a full Arch sync upgrade inside the update container and writes maintenance-repo commits as the unprivileged `builder` user to avoid ownership issues.

---

## Packaging notes

- PKGBUILD metadata declares `license=('MIT')` to match upstream.
- Packages copy license and notice files already present in the upstream Linux release artifact (`LICENSE*`, `LICENSES*`) into `/usr/share/licenses/$pkgname/` without synthesizing replacement license text.
- The app bundles `ffmpeg-static`; system `ffmpeg` is not a hard dependency.
- The upstream Linux wrapper forces XWayland under Wayland sessions for overlay positioning, so `libx11` remains a hard dependency even on Wayland systems.

You can also trigger the workflow manually via **Actions → Update AUR Packages → Run workflow**.

### Required secrets

| Secret | Description |
|---|---|
| `AUR_SSH_PRIVATE_KEY` | ed25519 private key registered on [aur.archlinux.org](https://aur.archlinux.org) (see setup below) |

### Setting up `AUR_SSH_PRIVATE_KEY`

#### Step 1 — Generate an SSH key pair

```sh
ssh-keygen -t ed25519 -C "aur-bot@openwhispr-aur" \
  -f ~/.ssh/aur_openwhispr -N ""
```

This creates:
- `~/.ssh/aur_openwhispr` — **private key** (goes to GitHub Secrets)
- `~/.ssh/aur_openwhispr.pub` — **public key** (goes to AUR)

#### Step 2 — Register the public key on AUR

1. Log in to <https://aur.archlinux.org>.
2. Go to **My Account → SSH Public Keys**.
3. Paste the contents of `~/.ssh/aur_openwhispr.pub` and save.

#### Step 3 — Add the private key as a GitHub secret

1. In this repository go to **Settings → Secrets and variables → Actions**.
2. Click **New repository secret**.
3. Name: `AUR_SSH_PRIVATE_KEY`
4. Value: full contents of `~/.ssh/aur_openwhispr` (including the `-----BEGIN …-----` header/footer).
5. Save.

---

## Repository layout

```
openwhispr-aur/
├── .github/workflows/
│   └── update-aur.yml            # Automated dual-upstream update workflow
├── openwhispr-bin/
│   ├── PKGBUILD
│   └── .SRCINFO
└── openwhispr-vulkan/
    ├── PKGBUILD
    └── .SRCINFO
```
