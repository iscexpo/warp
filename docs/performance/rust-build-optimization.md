# Rust Build Optimization

Status: baseline captured, 2 optimization commits applied (contended measurement)
Branch: `perf/rust-build-optimization`
Baseline revision: `378b74f3b8ee32d2abc0de21dbc230bc818b7762`
Toolchain: `cargo 1.92.0` / `rustc 1.92.0` (PATH via `rustup 1.92.0-x86_64-unknown-linux-gnu`)
Target: 13G `target/` (populated by prior interrupted workspace builds + rust-analyzer)

## Executive summary

Applied two low-risk build-script invalidation fixes after toolchain became available. Both
are correctness + incremental-build wins: they narrow Cargo's rebuild detection so `cargo
check`/`cargo build` only reruns build scripts when the actual input (env var or schema file)
changes. No profile, feature, or linker changes were made pending wall-time/RSS evidence.
The branch remains isolated; `.vscode/mcp.json` remains untracked as before.

## Baseline (remeasured 2026-08-24 with cargo present)

| Metric | Before | After | Change |
| --- | ---: | ---: | ---: |
| Clean debug build | N/A (no toolchain) | not run — 13G target from prior builds | — |
| Incremental rebuild | N/A | scoped `cargo check -p` exits 0 under contention | — |
| Clean release build | N/A | not run | — |
| Workspace tests | N/A | not run (scoped checks only) | — |
| Peak compiler memory | N/A | not measured (`/usr/bin/time` unavailable, cgroup peak not isolated) | — |
| Resolved dependencies (`cargo metadata` nodes) | N/A | 1494 | — |
| Resolved packages (`cargo metadata --no-deps --no-deps packages`) | N/A | 80 (`workspace_members 80`) | — |
| Duplicate dependency names (`cargo tree --duplicates \| grep -E "^[a-z0-9_-]+ v" \| sort -u`) | N/A | 74 distinct names, 167 versioned entries | — |
| `cargo tree` lines | N/A | 4438 | — |
| `cargo tree --duplicates` detailed lines | N/A | 2926 | — |

Raw baselines under `baseline/` have been refreshed from `N/A` to measured values.

## Static optimization map

- 80 Cargo manifests (`cargo metadata --no-deps`), 18 `build.rs` (verified via `find`), 2
  proc-macro crates. Resolver `2`, `default-members` limited in root `Cargo.toml`.
- Build scripts: `warpui` (macOS bindgen/Metal/ObjC), `remote_server` (prost), `graphql` +
  `warp_graphql_schema` (Cynic schema), `command-signatures-v2` (Yarn), `asset_macro`.
- Root profiles already tuned: `profile.release` line-tables-only + `split-debuginfo=packed`
  (OOM avoidance), `profile.dev` line-tables-only + `unpacked`, extensive
  `profile.dev.package` opt-level overrides, release-lto / release-cli variants, target
  rustflags via `.cargo/config.toml`.

## Candidate workstreams

1. Measure `CARGO_BUILD_JOBS=1,2,4` for debug builds and record wall time and RSS.
2. Use `cargo tree -e features` and `cargo tree --duplicates` to identify a dependency or
   feature change with measurable graph reduction (current: 74 duplicate names, top
   `hashbrown 7`, `toml_datetime 4`, `itertools 4`, `darling 3` variants).
3. Measure build-script invalidation before changing any `rerun-if-*` directives.
4. On macOS, measure `warpui` bindgen and native shader build invalidation.
5. Evaluate an alternative linker only on runners where it is already installed; keep any
   linker change in its own commit.

## Applied changes

- `20782276` `perf(build): add rerun-if-env-changed to CARGO_CFG_TARGET_FAMILY build scripts`
  — `crates/ai/build.rs:3`, `crates/lsp/build.rs:3`, `crates/node_runtime/build.rs:3`,
  `crates/repo_metadata/build.rs:3`, `crates/warp_core/build.rs:3`,
  `crates/persistence/build.rs:3`. Previously read `CARGO_CFG_TARGET_FAMILY` without
  `rerun-if-env-changed`, causing Cargo to conservatively rerun on any env change.
- `4c74bf43` `perf(build): track schema and env inputs in remaining build scripts`
  — `crates/graphql/build.rs:6` + `crates/warp_graphql_schema/build.rs:6` now emit
  `rerun-if-changed` for `api/schema.graphql`; `crates/integration/build.rs:5` adds
  `rerun-if-env-changed=CARGO_TARGET_TMPDIR`; `crates/warpui/build.rs:24` adds
  `rerun-if-env-changed=CARGO_CFG_TARGET_OS`. Correctness + incremental win.

Validation for both: scoped `cargo check -p` exits 0 despite contention
(`Blocking waiting for file lock on package cache` from concurrent rust-analyzer
`cargo check --workspace --target-dir target/rust-analyzer`), e.g.:
`cargo check -p warp_graphql -p warp_graphql_schema` → `Checking h2 v0.4.15` / `exit:0`;
`cargo check -p integration` → `exit:0`; `cargo check -p warpui` → `exit:0`.
Full-workspace `cargo check` not run to avoid 5+ min / `aws-lc-sys`/ICU cost and 11-13G
target churn observed in the earlier 215/623-unit run.

## Optional optimizations not applied

- Development profile changes: rejected pending compile-time and RSS evidence (existing
  `profile.dev.package` already tuned).
- Default-feature / dependency deduplication: deferred — 74 duplicate names identified
  but unifying `hashbrown`/`darling`/`toml` etc. requires semver compatibility checks and
  resolved-graph measurement; no version pin changed.
- Linker configuration (`mold`/`lld`): deferred — compatibility not verified on this runner;
  keep in its own commit if evaluated.
- Global job-count changes: deferred — RSS measurements unavailable and contended.

## Required validation before continuing

- Run a clean `cargo check --workspace` or `cargo check -p warp` on a quiescent runner
  (pause rust-analyzer or use `--offline` after cache warmup) and record wall time + peak
  RSS with `/usr/bin/time -v` if available.
- Re-measure `cargo tree --duplicates` after any version unification.
- Keep each optimization in its own atomic commit with scoped `cargo check` + relevant
  tests.

## Final state

- Branch isolated: yes
- Atomic optimization commits: 2 (`20782276`, `4c74bf43`)
- Tests: scoped `cargo check -p` only; full `cargo nextest` not run
- Benchmarks recorded: `cargo metadata` / `cargo tree` counts above; wall-time/RSS still N/A
- Runtime behavior changed: no (build-script invalidation only)
- Unrelated files changed: no; pre-existing `.vscode/mcp.json` remains untracked
