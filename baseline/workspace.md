# Workspace Baseline

Date: 2026-08-24 (remeasured 2026-08-25 with cargo 1.92.0)
Revision: `378b74f3b8ee32d2abc0de21dbc230bc818b7762` (branch `perf/rust-build-optimization`
now at `4c74bf43`)

## Structure

- Cargo manifests found: 80 (`cargo metadata --no-deps` `workspace_members 80`, `packages
  80`; `find crates -name Cargo.toml` also 80 including `app/Cargo.toml`)
- Build scripts found: 18 (`find crates -name build.rs` + `app/build.rs` = 19 total
  including app; crates-only 18)
- Proc-macro crates identified: 2 (`settings_value_derive`, `asset_macro` via static scan;
  confirmed in `cargo metadata`)
- Build scripts using bindgen: 1 (`crates/warpui/build.rs`)
- Build scripts using protobuf generation: 1 (`crates/remote_server/build.rs`)
- Build scripts registering Cynic schemas: 2 (`crates/graphql/build.rs`,
  `crates/warp_graphql_schema/build.rs`)
- Default members are explicitly limited in the root `Cargo.toml` (12 entries).
- Cargo resolver: 2
- `cargo tree` lines: 4438
- `cargo metadata` resolve nodes: 1494

## Existing build controls

The root manifest already uses line-table-only debug information, split debug
information (`profile.dev` `split-debuginfo=unpacked`, `profile.release`
`split-debuginfo=packed`), a tuned `[profile.dev.package]` (per-crate `opt-level=3`
for terminal/profile-sensitive crates), and dedicated `release-lto` / `release-cli` /
`release-tui` / `release-wasm` profiles. The repository config (`.cargo/config.toml`)
also sets target-specific Rust flags (`symbol-mangling-version=v0`) and uses the Git CLI
for Git dependencies (`net.git-fetch-with-cli = true`).

## Applied invalidation fixes (post-baseline)

- `20782276`: added `rerun-if-env-changed=CARGO_CFG_TARGET_FAMILY` to 6 crates
  (`ai`, `lsp`, `node_runtime`, `repo_metadata`, `warp_core`, `persistence`).
- `4c74bf43`: added schema `rerun-if-changed` to `graphql`/`warp_graphql_schema` and env
  tracking to `integration` (`CARGO_TARGET_TMPDIR`) and `warpui` (`CARGO_CFG_TARGET_OS`).

## Candidate hotspots (still present)

- `app`: largest application crate and primary binary; `app/build.rs` is the most
  complex build script (Metal, ObjC, Sentry, dock tile, Windows assets).
- `warpui`: macOS-only bindgen, Metal compilation, and Objective-C compilation.
- `remote_server`: protobuf code generation in `build.rs`.
- `graphql` and `warp_graphql_schema`: schema registration — now correctly tracked.
- `command-signatures-v2`: invokes Yarn from a build script and tracks generated JS.
- `asset_macro`: proc macro with high recursion limit and asset path work.
- Duplicate dependencies: 74 distinct names duplicated (top `hashbrown 7`,
  `toml_datetime 4`, `itertools 4`, `darling 3`).

These are candidates for measurement, not approved optimizations.
