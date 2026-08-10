# AGENTS.md — workmux

Rust CLI tool (edition 2024) for orchestrating git worktrees + tmux. Package name on crates.io is `workmux`; the local clone directory is `qual`.

## Build & dev commands

All commands run via **[just](https://github.com/casey/just)**. The project uses **[checkle](https://github.com/raine/checkle)** as a unified check runner.

```bash
just check       # Run ALL checks: rustfmt, clippy, ruff, pyright, unit tests, docs
just check-ci    # Same as check + fail if working tree is dirty
just test        # Unit tests only (cargo test --bin workmux)
just itest       # Integration tests (pytest, parallel). Requires tmux + tests/venv/
just itest tests/test_workmux_add/test_basic.py  # Run a single integration test file
just build       # cargo build
just clippy      # cargo clippy --all-targets
just format      # rustfmt + ruff format
just install-dev # Symlink debug binary to ~/.cargo/bin/workmux
```

The pre-commit hook runs `checkle pre-commit`. The pre-merge hook (configured in `.workmux.yaml`) runs `just check-ci` then `just itest`.

## Testing

### Integration tests (Python/pytest)

Integration tests live in `tests/` and require:

1. **tmux** installed and NOT nested inside another tmux session
2. **Python venv** at `tests/venv/` (install: `python -m venv tests/venv && source tests/venv/bin/activate && pip install -r tests/requirements.txt`)
3. **Binary built**: `cargo build` first
4. Multiple shells tested: bash, zsh, fish, nu (skip missing with `TEST_SHELL=fish`)
5. Optional: WezTerm backend (`just itest --backend=wezterm`)

Tests create isolated tmux servers via private sockets. They use a session-scoped template git repo to avoid per-test git init overhead.

**Key fixtures** (see `tests/conftest.py`):
- `repo_path` — isolated git repo (copies from session template)
- `mux_server` — parameterized backend (tmux or wezterm)
- `workmux_exe_path` — path to `target/debug/workmux`

## Architecture

Single Rust binary (not a workspace). Entrypoint: `src/main.rs` → `cli::run()`.

Key source modules:
- `src/cli.rs` — Clap command definitions (~1500 lines)
- `src/config.rs` — `.workmux.yaml` parsing and config resolution (global → project)
- `src/git/` — Git operations (worktrees, branches, remotes)
- `src/multiplexer/` — Backend abstraction (tmux, wezterm, kitty, zellij)
- `src/workflow/` — High-level commands: add, merge, remove, rebase
- `src/command/` — Command execution and argument types
- `src/state/` — Agent state tracking (pid, status)
- `src/sandbox/` — Container/Lima sandbox support
- `src/ui/` — Dashboard TUI (ratatui + crossterm)
- `src/agent_setup/` — Agent hook/skill installation (`workmux setup`)

Other notable directories:
- `tests/` — Python pytest integration tests
- `docs/` — Astro documentation site (bun)
- `scripts/` — Shell scripts (install, CI helpers, cleanup)
- `skills/` — Shipped agent skills for `/worktree`, `/merge`, etc.
- `.claude-plugin/` — Claude Code status-tracking plugin
- `resources/` — OpenCode, Gemini plugin resources

## Code patterns

- **Error handling**: `anyhow::Result` throughout, `thiserror` for custom error types
- **Logging**: `tracing` crate with env-filter (`RUST_LOG`); `logger.rs` handles init
- **CLI**: `clap` with derive macros; custom `TypedValueParser` impls for shell completions
- **Command execution**: `src/cmd.rs` `Cmd` builder wraps `std::process::Command` with unified error context
- **No barrel exports**: Each module re-exports explicitly; no `pub use *`
- **No path aliases** in Cargo.toml

## Configuration files

| File | Purpose |
|---|---|
| `.workmux.yaml` | Project-level workmux config (panes, hooks, file ops) |
| `~/.config/workmux/config.yaml` | Global workmux config |
| `checkle.toml` | Check definitions: rustfmt, clippy, ruff, pyright, unit tests, docs |
| `deny.toml` | cargo-deny license/advisory/bans config |
| `devbox.json` | Reproducible dev environment (rustup, just, ruff) |
| `justfile` | Task runner recipes |
| `flake.nix` / `flake.lock` | Nix flake for reproducible builds |
| `.github/workflows/ci.yml` | CI: two jobs — `checks` (rust+cargo check) and `python-tests` (pytest+tmux) |

## Gotchas

- **CI vs local**: `just check-ci` fails if checks produce uncommitted changes. Run `just check` locally first to avoid CI-only failures.
- **Integration tests need a clean tmux**: Don't run `just itest` from inside a tmux session. The test framework creates its own isolated tmux server.
- **Symlinked tests/venv**: The repo's `.workmux.yaml` symlinks `tests/venv` for faster worktree creation. If you add new pip dependencies, update the main worktree venv then run `workmux sync-files --all`.
- **cargo install --locked**: The `just install` recipe uses `--offline --locked`. Use `just install-dev` for development.
- **Windows not supported**: This tool targets Unix (macOS/Linux). Uses `nix`, `libc`, `signal-hook`, platform-specific notification crates.
- **Rust edition 2024**: Uses newer Rust features. Ensure your toolchain is up to date via `rustup`.
- **docs/ uses bun**: The documentation site in `docs/` requires bun (not npm/pnpm). Run `cd docs && bun install` then `bun run dev`.
- **The repo directory is `qual`** but the binary/package is `workmux`. Don't get confused by the directory name.
- **Pre-merge runs check-ci + itest**: When merging via `workmux merge`, the pre-merge hook runs `just check-ci` then `just itest`. A failure blocks the merge.
