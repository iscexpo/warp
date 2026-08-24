# Workspace Baseline

Date: 2026-08-24
Revision: `378b74f3b8ee32d2abc0de21dbc230bc818b7762`

## Structure

- Cargo manifests found: 81
- Build scripts found: 18
- Proc-macro crates identified statically: 2
- Build scripts using bindgen: 1 (`crates/warpui/build.rs`)
- Build scripts using protobuf generation: 1 (`crates/remote_server/build.rs`)
- Build scripts registering Cynic schemas: 2
- Default members are explicitly limited in the root `Cargo.toml`.
- Cargo resolver: 2

Exact workspace package/member and dependency counts are N/A because
`cargo metadata --no-deps` could not run in this container.

## Existing build controls

The root manifest already uses line-table-only debug information, split debug
information, a tuned `[profile.dev.package]`, and dedicated release/LTO profiles.
The repository config also sets target-specific Rust flags and uses the Git CLI
for Git dependencies.

## Candidate hotspots

- `app`: largest application crate and the primary binary target.
- `warpui`: macOS-only bindgen, Metal compilation, and Objective-C compilation.
- `remote_server`: protobuf code generation in `build.rs`.
- `graphql` and `warp_graphql_schema`: schema registration in separate build scripts.
- `command-signatures-v2`: invokes Yarn from a build script and tracks generated JS.
- `asset_macro`: proc macro with a high recursion limit and asset path work.

These are candidates for measurement, not approved optimizations.
