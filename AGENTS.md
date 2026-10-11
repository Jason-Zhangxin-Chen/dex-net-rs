# AGENTS.md

Guidance for AI coding agents (Claude Code, Codex, Cursor, etc.) working in this repository.

## Project overview

`dex-net-rs` is a Rust workspace for a modern blockchain network with built-in DEX
primitives as first-class citizens. The design spans a QUIC-based P2P network layer, an
execution layer with native exchange primitives (CLOB order books, asset primitives, a
QMDB-backed state engine), and a Minimmit BFT consensus layer. `README.md` holds the
component-level design rationale; this file covers how to work on the code.

The project is in early development: the workspace currently contains a single `crypto`
crate stub, with the documentation and CI scaffolding built out ahead of the core layers.

## Workspace layout

```
dex-net-rs/
├── crates/              # workspace members
│   └── crypto/          # foundational crypto crate (currently a stub)
├── .github/workflows/   # CI (ci.yml: build+test, clippy, fmt, cargo-deny)
├── Cargo.toml           # workspace manifest (members = ["crates/crypto"])
├── Cargo.lock           # committed; owned by dependabot bumps
├── clippy.toml          # lint thresholds and domain-term whitelist
├── rustfmt.toml         # formatting: max_width 100, module reordering
├── deny.toml            # dependency policy: sources, licenses, advisories
├── rust-toolchain.toml  # pinned Rust 1.94.1 (edition 2024)
├── README.md            # design overview of the network layers
└── AGENTS.md            # agent guidance for all coding agents (this file)
```

## Architecture

Four layers, described in detail in `README.md`:

- **P2P network layer** — QUIC-based transport with NAT traversal, peer discovery, and
  message routing (inspired by iroh).
- **Execution layer** — structured, latency-aware gossiping; erasure-coded (Reed–Solomon)
  block propagation; Freezer + chain databases; an optional VM; a deterministic state
  engine built on QMDB; and native asset & exchange primitives (CLOB, RFQ, order types).
- **Consensus layer** — Minimmit BFT (80% quorum, fast finality) with consensus messaging
  and BLS12-381 signature aggregation.
- **Core crate boundary** — the core crate must stay executor-agnostic: no tokio types in
  public signatures (enforced via `#[deny(clippy::disallowed_types)]`).

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
