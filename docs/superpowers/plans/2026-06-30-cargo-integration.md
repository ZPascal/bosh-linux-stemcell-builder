# Cargo Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `bosh_rust` stage that installs Cargo and the Rust toolchain (via rustup) into `/var/vcap/bosh` inside the stemcell OS image, making `cargo` and `rustc` available at runtime on deployed VMs.

**Architecture:** A new stage `stemcell_builder/stages/bosh_rust/` follows the existing two-file (`config.sh` + `apply.sh`) pattern. It runs after `base_ubuntu_build_essential` (which provides `clang` and `build-essential` as required C linker deps) in `ubuntu_os_stages`. rustup installs stable Rust into `/var/vcap/bosh/rustup` and `/var/vcap/bosh/cargo`; `cargo` and `rustc` are symlinked into `/var/vcap/bosh/bin/`.

**Tech Stack:** Bash, chroot environment via `run_in_chroot`, rustup installer (`sh.rustup.rs`), RSpec/Serverspec for OS image tests, Ruby `stage_collection.rb` for pipeline wiring.

---

## File Map

| Action | Path | Responsibility |
|--------|------|---------------|
| Create | `stemcell_builder/stages/bosh_rust/config.sh` | Minimal config stage boilerplate |
| Create | `stemcell_builder/stages/bosh_rust/apply.sh` | Download rustup, install stable Rust, symlink binaries |
| Modify | `bosh-stemcell/lib/bosh/stemcell/stage_collection.rb:282` | Register `:bosh_rust` after `:base_ubuntu_build_essential` |
| Modify | `bosh-stemcell/spec/os_image/ubuntu_spec.rb:549` | Add `describe "bosh_rust"` assertions |

---

## Task 1: Write the failing spec for `bosh_rust`

**Files:**
- Modify: `bosh-stemcell/spec/os_image/ubuntu_spec.rb:549`

- [ ] **Step 1: Open the spec file and locate the insertion point**

  The file ends at line 550. The closing `end` of the top-level `describe` block is at line 549. Insert the new describe block just before line 549 (before the final `end`).

- [ ] **Step 2: Add the failing spec block**

  In `bosh-stemcell/spec/os_image/ubuntu_spec.rb`, insert before the final `end` on line 549:

  ```ruby
  describe "bosh_rust" do
    describe file("/var/vcap/bosh/bin/cargo") do
      it { should be_file }
      it { should be_executable }
    end

    describe file("/var/vcap/bosh/bin/rustc") do
      it { should be_file }
      it { should be_executable }
    end

    describe file("/var/vcap/bosh/cargo") do
      it { should be_directory }
    end

    describe file("/var/vcap/bosh/rustup") do
      it { should be_directory }
    end
  end
  ```

- [ ] **Step 3: Verify the spec file is syntactically valid**

  ```bash
  cd bosh-stemcell && bundle exec rspec spec/os_image/ubuntu_spec.rb --dry-run 2>&1 | tail -5
  ```

  Expected: no syntax errors; the run completes with something like `0 examples, 0 failures` (dry run, no real OS image needed).

- [ ] **Step 4: Commit the failing spec**

  ```bash
  git add bosh-stemcell/spec/os_image/ubuntu_spec.rb
  git commit -m "test: add failing spec for bosh_rust stage"
  ```

---

## Task 2: Create `stemcell_builder/stages/bosh_rust/config.sh`

**Files:**
- Create: `stemcell_builder/stages/bosh_rust/config.sh`

- [ ] **Step 1: Create the stage directory**

  ```bash
  mkdir -p stemcell_builder/stages/bosh_rust
  ```

- [ ] **Step 2: Write `config.sh`**

  Create `stemcell_builder/stages/bosh_rust/config.sh` with this content:

  ```bash
  #!/usr/bin/env bash
  set -e

  base_dir=$(readlink -nf "$(dirname "${0}")/../..")
  source "${base_dir}/lib/prelude_config.bash"
  ```

  This is the minimal config boilerplate — no assets to download in this stage (rustup is fetched inside the chroot at apply time).

- [ ] **Step 3: Make it executable**

  ```bash
  chmod +x stemcell_builder/stages/bosh_rust/config.sh
  ```

- [ ] **Step 4: Commit**

  ```bash
  git add stemcell_builder/stages/bosh_rust/config.sh
  git commit -m "feat: add bosh_rust/config.sh stage skeleton"
  ```

---

## Task 3: Create `stemcell_builder/stages/bosh_rust/apply.sh`

**Files:**
- Create: `stemcell_builder/stages/bosh_rust/apply.sh`

- [ ] **Step 1: Write `apply.sh`**

  Create `stemcell_builder/stages/bosh_rust/apply.sh` with this content:

  ```bash
  #!/usr/bin/env bash
  set -e

  base_dir=$(readlink -nf "$(dirname "${0}")/../..")
  source "${base_dir}/lib/prelude_apply.bash"

  curl_five_times "${chroot}/tmp/rustup-init.sh" "https://sh.rustup.rs"

  run_in_chroot "${chroot}" "
    chmod +x /tmp/rustup-init.sh
    RUSTUP_HOME=/var/vcap/bosh/rustup \
    CARGO_HOME=/var/vcap/bosh/cargo \
    /tmp/rustup-init.sh -y --no-modify-path --default-toolchain stable
    rm /tmp/rustup-init.sh
    ln -sf /var/vcap/bosh/cargo/bin/cargo /var/vcap/bosh/bin/cargo
    ln -sf /var/vcap/bosh/cargo/bin/rustc /var/vcap/bosh/bin/rustc
  "
  ```

  Key points:
  - `curl_five_times` is defined in `lib/helpers.sh` — it retries up to 5 times and exits non-zero if the file was never created.
  - `RUSTUP_HOME` and `CARGO_HOME` are set so the toolchain lands under `/var/vcap/bosh/`, not under a home directory.
  - `--no-modify-path` prevents rustup from touching shell profiles inside the chroot.
  - `ln -sf` creates symlinks (force-overwrite if somehow already present) so `cargo` and `rustc` appear in the BOSH-managed bin dir.

- [ ] **Step 2: Make it executable**

  ```bash
  chmod +x stemcell_builder/stages/bosh_rust/apply.sh
  ```

- [ ] **Step 3: Commit**

  ```bash
  git add stemcell_builder/stages/bosh_rust/apply.sh
  git commit -m "feat: add bosh_rust/apply.sh — install Rust via rustup into /var/vcap/bosh"
  ```

---

## Task 4: Register `:bosh_rust` in the stage pipeline

**Files:**
- Modify: `bosh-stemcell/lib/bosh/stemcell/stage_collection.rb:282`

- [ ] **Step 1: Open `stage_collection.rb` and find `ubuntu_os_stages`**

  The method starts at line 277. The `:base_ubuntu_build_essential` symbol is at line 282.

- [ ] **Step 2: Insert `:bosh_rust` after `:base_ubuntu_build_essential`**

  Change:
  ```ruby
        :base_ubuntu_build_essential,
        :base_ubuntu_packages,
  ```

  To:
  ```ruby
        :base_ubuntu_build_essential,
        :bosh_rust,
        :base_ubuntu_packages,
  ```

- [ ] **Step 3: Verify Ruby syntax**

  ```bash
  cd bosh-stemcell && bundle exec ruby -e "require_relative 'lib/bosh/stemcell/stage_collection'" && echo "OK"
  ```

  Expected: `OK`

- [ ] **Step 4: Run the existing unit tests**

  ```bash
  cd bosh-stemcell && bundle exec rspec spec/ --exclude-pattern "spec/os_image/**,spec/stemcells/**" 2>&1 | tail -10
  ```

  Expected: all examples pass, 0 failures.

- [ ] **Step 5: Commit**

  ```bash
  git add bosh-stemcell/lib/bosh/stemcell/stage_collection.rb
  git commit -m "feat: register bosh_rust stage in ubuntu_os_stages pipeline"
  ```

---

## Task 5: Verify end-to-end (optional local smoke test)

> This task requires a Linux host with Docker or a working `chroot`/`unshare` environment (the build system uses `run_in_chroot` which calls `unshare -f -p -m`). On macOS this stage cannot be run directly — it is verified by CI.

- [ ] **Step 1: Confirm all new files exist and are executable**

  ```bash
  ls -la stemcell_builder/stages/bosh_rust/
  ```

  Expected:
  ```
  -rwxr-xr-x  apply.sh
  -rwxr-xr-x  config.sh
  ```

- [ ] **Step 2: Confirm stage is wired into the pipeline**

  ```bash
  cd bosh-stemcell && bundle exec ruby -e "
    require_relative 'lib/bosh/stemcell/stage_collection'
    require_relative 'lib/bosh/stemcell/definition'
    d = Bosh::Stemcell::Definition.for('aws', 'xen-hvm', 'ubuntu', 'jammy')
    c = Bosh::Stemcell::StageCollection.new(d)
    stages = c.operating_system_stages
    idx = stages.index(:bosh_rust)
    build_essential_idx = stages.index(:base_ubuntu_build_essential)
    puts \"bosh_rust at index: #{idx}\"
    puts \"base_ubuntu_build_essential at index: #{build_essential_idx}\"
    puts \"bosh_rust comes after base_ubuntu_build_essential: #{idx > build_essential_idx}\"
  "
  ```

  Expected:
  ```
  bosh_rust at index: 4
  base_ubuntu_build_essential at index: 3
  bosh_rust comes after base_ubuntu_build_essential: true
  ```
  (Exact indices may vary; what matters is `bosh_rust` immediately follows `base_ubuntu_build_essential` and the boolean is `true`.)

- [ ] **Step 3: Confirm full test suite is green**

  ```bash
  cd bosh-stemcell && bundle exec rspec spec/ --exclude-pattern "spec/os_image/**,spec/stemcells/**" 2>&1 | tail -5
  ```

  Expected: 0 failures.
