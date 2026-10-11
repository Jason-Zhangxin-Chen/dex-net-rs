# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

## Workspace layout

## Architecture

## Development rules

### Toolchain

`rust-toolchain.toml` pins Rust **1.94.1** (first stable of edition 2024, which every crate
uses). CI is pinned to the same version — keep them in lockstep; do not bump one without
updating the other. `Cargo.lock` is committed; dependabot (weekly, Monday 09:00 UTC) owns
dependency bumps with grouped patch/minor updates.

### CI gates (`.github/workflows/ci.yml`)

The single required check `all-checks` composes four jobs — every PR must pass all of:

1. Build & test: `cargo build --workspace --locked` and `cargo nextest run --workspace`
   (CI sets `RUSTFLAGS="-D warnings"` — warnings fail the build)
2. Clippy: `cargo clippy --workspace --all-targets --locked -- -D warnings`
3. Format: `cargo fmt --all -- --check`
4. Dependency audit: `cargo deny check`

### Lint and format configuration

- `clippy.toml`: `cognitive-complexity-threshold = 30`, `type-complexity-threshold = 400`,
  and a `doc-valid-idents` whitelist of domain terms (DEX, Redpanda, OrderBook, …). Add new
  domain terms there rather than sprinkling `#[allow]` on doc lints.
- `rustfmt.toml`: `max_width = 100`, imports/modules reordered, `use_small_heuristics = "Max"`,
  Unix newlines.
- The core crate must stay **executor-agnostic**: no tokio types in public signatures
  (documented in `clippy.toml`; enforced via `#[deny(clippy::disallowed_types)]`).

### Dependency policy (`deny.toml`)

- Sources: crates.io only — git and unknown registries are denied; wildcard versions denied.
- Licenses: permissive / weak-copyleft allow-list (MIT, Apache-2.0, BSD, BSL-1.0, ISC,
  MPL-2.0, …) — new dependencies must fit; `ring` is clarified as `MIT AND ISC AND OpenSSL`.
- Advisories: yanked versions denied, `unsound = all`, `unmaintained = workspace`.

## Conventions

- **Commit messages**: lowercase `<type-or-scope>, <description>`, e.g.
  `feature, zero value definitions for value types.` /
  `refine, cache line friendly structure design for book.` /
  `lint, fix fmt errors.` / `test, fix test.` / `ci, target to main.`
- **Workflow**: feature branch off `main` → PR (CI also runs on merge queues). Branches are
  named for the work (`primitives`, `serde`, `book`, `visibility`, `core`, `impl`).
- **Tests**: inline `#[cfg(test)] mod tests` in the same source file, heavy on MessagePack
  round-trips and wire-format assertions. CI runs nextest; plain `cargo test` works locally.
- **Performance principles**: cache-line-friendly hot/cold splits, lock-free match path,
  no `Copy` for large types, non-blocking listeners. When touching hot structures, preserve
  the layout discipline — there is commit history dedicated to it.