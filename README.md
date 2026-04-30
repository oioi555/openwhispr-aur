# openwhispr-aur

AUR maintenance repository for two [OpenWhispr](https://github.com/OpenWhispr/openwhispr) packages.

| Package | Description |
|---|---|
| [`openwhispr-bin`](./openwhispr-bin/) | Pre-built binary from the upstream tar.gz release. Vulkan is an optional dependency. |
| [`openwhispr-vulkan`](./openwhispr-vulkan/) | Same binary, but `vulkan-icd-loader` is a **hard** dependency — use this on AMD/Intel/NVIDIA systems where you want GPU-accelerated on-device Whisper/Parakeet inference. |

Both packages `provides=('openwhispr')` and `conflicts` with each other, so only one may be installed at a time.

---

## Install

```sh
# Standard binary (Vulkan optional)
paru -S openwhispr-bin

# Vulkan-first variant (Vulkan required)
paru -S openwhispr-vulkan
```

---

## Automated updates (GitHub Actions)

The workflow in [`.github/workflows/update-aur.yml`](.github/workflows/update-aur.yml) runs **daily at 08:00 UTC** and:

1. Queries the GitHub Releases API for the latest `OpenWhispr/openwhispr` tag.
2. Compares it with the `pkgver` tracked in this repository.
3. If a new version is available:
   - Updates `pkgver` and resets `pkgrel=1` in both PKGBUILDs.
   - Runs `updpkgsums` to recalculate SHA-256 checksums.
   - Regenerates `.SRCINFO` via `makepkg --printsrcinfo`.
   - Pushes the updated files to each AUR repository via SSH.
   - Commits the updated files back to this maintenance repository.

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
│   └── update-aur.yml        # Automated update workflow
├── openwhispr-bin/
│   ├── PKGBUILD
│   └── .SRCINFO
└── openwhispr-vulkan/
    ├── PKGBUILD
    └── .SRCINFO
```