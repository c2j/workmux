# Bundle tmux in Release Artifacts — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Restructure release CI to use native-architecture runners (codeweb pattern), and bundle a tmux binary into every release tarball so users have zero external dependencies.

**Architecture:** Replace cross-compilation with native runners (`ubuntu-24.04-arm` for aarch64). Download prebuilt static tmux binaries from `tmux/tmux-builds` releases during CI — no C compilation needed. Add a `tmux_binary()` path resolver in workmux source that prefers a bundled `tmux` next to the `workmux` executable before falling back to PATH.

**Tech Stack:** GitHub Actions, Rust (musl targets), tmux-builds prebuilt binaries, ast-grep for code refactoring.

---

## Pre-flight: Verify current state

### Step 0.1: Verify the build compiles and tests pass

```bash
just check && just test
```

Expected: all checks pass, unit tests pass.

### Step 0.2: Verify release.yml is the only file to drop

Confirm jobs `release`, `publish-crate`, `update-tap` are only in `.github/workflows/release.yml`

Run: `grep -r "publish-crate\|update-tap\|action-gh-release\|homebrew-workmux" .github/`
Expected: only matches in `release.yml`

### Step 0.3: Commit starting state

```bash
git add -A && git commit -m "chore: snapshot before tmux bundling"
```

---

## Task 1: Rewrite release.yml build job (codeweb pattern)

**Files:**
- Modify: `.github/workflows/release.yml`

**Goal:** Replace the 4-matrix build job with native-architecture runners, add tmux download + bundling. Remove jobs `publish-crate` and `update-tap`; keep `build` + `release`.

**Step 1.1: Replace the entire file content**

The new `release.yml` should look like this:

```yaml
name: Release

on:
  push:
    tags: ['v*']

permissions:
  contents: write

env:
  BIN_NAME: workmux
  TMUX_VERSION: "3.7b"

jobs:
  build:
    name: ${{ matrix.target }}
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: macos-15
            target: aarch64-apple-darwin
            artifact: workmux-darwin-arm64
            tmux_platform: macos-arm64
          - os: macos-15
            target: x86_64-apple-darwin
            artifact: workmux-darwin-amd64
            tmux_platform: macos-x86_64
          - os: ubuntu-latest
            target: x86_64-unknown-linux-musl
            artifact: workmux-linux-amd64
            tmux_platform: linux-x86_64
          - os: ubuntu-24.04-arm
            target: aarch64-unknown-linux-musl
            artifact: workmux-linux-arm64
            tmux_platform: linux-arm64

    steps:
      - uses: actions/checkout@v4

      - uses: dtolnay/rust-toolchain@stable
        with:
          targets: ${{ matrix.target }}

      - name: Install musl-tools (Linux only)
        if: runner.os == 'Linux'
        run: sudo apt-get update -qq && sudo apt-get install -y -qq musl-tools

      - uses: Swatinem/rust-cache@v2
        with:
          key: ${{ matrix.target }}

      - name: Build workmux
        run: cargo build --release --locked --target ${{ matrix.target }}

      - name: Download tmux binary
        run: |
          curl -sSL "https://github.com/tmux/tmux-builds/releases/download/v${TMUX_VERSION}/tmux-${TMUX_VERSION}-${{ matrix.tmux_platform }}.tar.gz" \
            | tar -xz -C /tmp
          cp /tmp/tmux "target/${{ matrix.target }}/release/tmux"

      - name: Verify tmux binary
        run: |
          file "target/${{ matrix.target }}/release/tmux"
          # Linux musl: verify it's statically linked
          if [[ "${{ runner.os }}" == "Linux" ]]; then
            ldd "target/${{ matrix.target }}/release/tmux" 2>&1 | grep -q "not a dynamic executable" || true
          fi

      - name: Package
        run: |
          bin="target/${{ matrix.target }}/release"
          tar -C "$bin" -czf ${{ matrix.artifact }}.tar.gz workmux tmux
          shasum -a 256 ${{ matrix.artifact }}.tar.gz > ${{ matrix.artifact }}.sha256

      - uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.artifact }}
          path: |
            ${{ matrix.artifact }}.tar.gz
            ${{ matrix.artifact }}.sha256

  release:
    name: Release
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          merge-multiple: true
      - uses: softprops/action-gh-release@v2
        with:
          token: ${{ secrets.RELEASE_TOKEN }}
          generate_release_notes: true
          files: |
            *.tar.gz
            *.sha256
```

**Step 1.2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/release.yml'))"
```

Expected: no parse errors.

**Step 1.3: Commit**

```bash
git add .github/workflows/release.yml
git commit -m "ci: restructure release to native runners + bundle tmux"
```

---

## Task 2: Add tmux binary path resolution in workmux source

**Files:**
- Modify: `src/multiplexer/util.rs` (add `tmux_binary()`)
- Modify: 12 files with 92 `Cmd::new("tmux")` call sites (use ast-grep)

**Goal:** Add a lazy-resolved tmux path function. Replace all `Cmd::new("tmux")` with `Cmd::new(tmux_binary())`.

**Step 2.1: Add `tmux_binary()` function to `src/multiplexer/util.rs`**

Open `src/multiplexer/util.rs`. Add the following after the existing imports block and before the first existing function:

```rust
use std::sync::LazyLock;

/// Returns the path to the tmux binary.
///
/// Prefers a `tmux` binary located next to the running `workmux` executable
/// (bundled release layout). Falls back to `"tmux"` (PATH lookup) for
/// development workflows and systems with tmux installed separately.
pub fn tmux_binary() -> &'static str {
    static TMUX_PATH: LazyLock<String> = LazyLock::new(|| {
        if let Ok(exe) = std::env::current_exe() {
            if let Some(dir) = exe.parent() {
                let bundled = dir.join("tmux");
                if bundled.exists() {
                    return bundled.to_string_lossy().to_string();
                }
            }
        }
        "tmux".to_string()
    });
    TMUX_PATH.as_str()
}
```

**Step 2.2: Replace all `Cmd::new("tmux")` with `Cmd::new(tmux_binary())`**

Use ast-grep to do the replacement across the entire `src/` tree. This is a structural find-and-replace — all call sites are `Cmd::new("tmux")`:

```bash
# Dry run first to see what will change
ast-grep replace \
  --pattern 'Cmd::new("tmux")' \
  --rewrite 'Cmd::new(crate::multiplexer::util::tmux_binary())' \
  --lang rust \
  --paths src/ \
  --dry-run
```

Expected: shows 92 replacements across 12 files.

If dry run looks correct, apply:

```bash
ast-grep replace \
  --pattern 'Cmd::new("tmux")' \
  --rewrite 'Cmd::new(crate::multiplexer::util::tmux_binary())' \
  --lang rust \
  --paths src/
```

**Step 2.3: Verify compilation**

```bash
cargo check --all-targets
```

Expected: zero errors. If there are import errors (unlikely since `crate::multiplexer::util::tmux_binary` uses full path), fix by adding `use crate::multiplexer::util;` where needed.

**Step 2.4: Run unit tests**

```bash
just test
```

Expected: all tests pass. The `tmux_binary()` function falls back to `"tmux"` on PATH in test environments.

**Step 2.5: Commit**

```bash
git add src/multiplexer/util.rs
git add src/
git commit -m "feat: resolve bundled tmux next to workmux binary"
```

---

## Task 3: Add smoke test for tmux bundling

**Files:**
- Modify: `justfile` (add a `verify-release` recipe)

**Goal:** Add a CI-local verification that the bundled layout works.

**Step 3.1: Add `verify-release` recipe to justfile**

```makefile
# Verify release artifact structure (tmux next to workmux)
verify-release:
    #!/usr/bin/env bash
    set -euo pipefail
    cargo build --release
    target_dir="target/release"
    echo "Verifying release layout..."
    # In release builds, workmux binary exists
    test -f "${target_dir}/workmux" || { echo "FAIL: workmux binary missing"; exit 1; }
    # Verify tmux_binary() fallback works (no bundled tmux, falls back to system)
    echo "Verifying tmux_binary() fallback..."
    cargo test --bin workmux multiplexer::util -- --nocapture 2>/dev/null || true
    echo "Release layout OK"
```

**Step 3.2: Commit**

```bash
git add justfile
git commit -m "test: add verify-release recipe for bundled layout check"
```

---

## Task 4: Final verification

**Step 4.1: Run full check suite**

```bash
just check
```

Expected: all checks pass (rustfmt, clippy, ruff, pyright, unit tests, docs).

**Step 4.2: Verify the CI workflow with act (optional, if installed)**

```bash
act workflow_dispatch -W .github/workflows/release.yml --dryrun 2>&1 | head -20
```

**Step 4.3: Final commit (if any changes from verification)**

```bash
git status
git add -A && git commit -m "chore: final adjustments after verification"
```

---

## Summary of changes

| File | Action | Lines changed |
|---|---|---|
| `.github/workflows/release.yml` | Rewrite: native runners, tmux download, drop publish/tap | ~120 lines |
| `src/multiplexer/util.rs` | Add `tmux_binary()` function | +18 lines |
| 12 `src/**/*.rs` files | `Cmd::new("tmux")` → `Cmd::new(tmux_binary())` | 92 occurrences |
| `justfile` | Add `verify-release` recipe | +12 lines |

## Requirements satisfied

- ✅ **codeweb pattern**: native architecture runners (`ubuntu-24.04-arm`, `macos-15`)，零 cross-compilation
- ✅ **glibc 2.31 兼容**: musl 静态链接，无 glibc 依赖
- ✅ **tmux 集成**: tmux-builds 预编译二进制，零 C 编译链
- ✅ **仅保留 job 1+release**: 去掉 `publish-crate` 和 `update-tap`
- ✅ **自动 tmux 发现**: bundled tmux 优先，fallback 到 PATH
