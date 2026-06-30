# Cargo Integration Design

**Date:** 2026-06-30
**Branch:** ubuntu-jammy

## Goal

Add Cargo (Rust package manager) and the Rust toolchain as a runtime dependency in the BOSH Linux stemcell OS image, so BOSH jobs deployed on VMs built from this stemcell can invoke `cargo` and `rustc` directly.

## Architecture

A new stage `bosh_rust` is added to `stemcell_builder/stages/`, following the standard two-file layout (`config.sh` + `apply.sh`). It is registered in `stage_collection.rb` inside `ubuntu_os_stages`, immediately after `:base_ubuntu_build_essential` (which installs `clang` and `build-essential` — required by rustup as a C linker).

## Components

| File | Purpose |
|------|---------|
| `stemcell_builder/stages/bosh_rust/config.sh` | Minimal config stage; no assets needed |
| `stemcell_builder/stages/bosh_rust/apply.sh` | Downloads rustup installer and installs stable Rust toolchain inside the chroot |
| `bosh-stemcell/lib/bosh/stemcell/stage_collection.rb` | Registers `:bosh_rust` after `:base_ubuntu_build_essential` in `ubuntu_os_stages` |

## Data Flow

During `stemcell:build_os_image`, the stage runner executes `bosh_rust/apply.sh` inside the chroot:

1. `curl_five_times` fetches `https://sh.rustup.rs` into `/tmp/rustup-init.sh` inside the chroot
2. `rustup-init` runs non-interactively (`-y --no-modify-path --default-toolchain stable`) with:
   - `RUSTUP_HOME=/var/vcap/bosh/rustup`
   - `CARGO_HOME=/var/vcap/bosh/cargo`
3. Symlinks `cargo` and `rustc` from `/var/vcap/bosh/cargo/bin/` into `/var/vcap/bosh/bin/`
4. Installer script removed from `/tmp`

At runtime on deployed VMs, `cargo` and `rustc` are available at `/var/vcap/bosh/bin/cargo` and `/var/vcap/bosh/bin/rustc`.

## Error Handling

- `set -e` aborts the build on any failure
- `curl_five_times` retries the download up to 5 times (consistent with blobstore CLI fetching)
- `rustup-init -y` exits non-zero on install failure, propagating through `set -e`
- Installer script is removed from `/tmp` after install

## Testing

New `describe "bosh_rust"` block in `bosh-stemcell/spec/os_image/ubuntu_spec.rb`:

- `/var/vcap/bosh/bin/cargo` exists and is executable
- `/var/vcap/bosh/bin/rustc` exists and is executable
- `/var/vcap/bosh/cargo` directory exists
- `/var/vcap/bosh/rustup` directory exists

## Decisions

- **rustup over apt**: Ubuntu Jammy ships an older Rust; rustup provides latest stable and allows future toolchain updates
- **`/var/vcap/bosh` prefix**: Consistent with blobstore CLIs and other BOSH-managed binaries; isolated from system paths
- **No rustup in final PATH**: `--no-modify-path` keeps rustup from modifying shell profiles; explicit symlinks into `/var/vcap/bosh/bin/` are the only PATH exposure
- **No pinned rustup installer checksum**: rustup verifies downloaded toolchains via its own signing chain; the installer itself is served over TLS from the canonical `sh.rustup.rs`
