# Rust Build Optimization

Status: baseline captured, optimization changes deferred
Branch: `perf/rust-build-optimization`
Baseline revision: `378b74f3b8ee32d2abc0de21dbc230bc818b7762`

## Executive summary

No optimization was applied because the current Ubuntu container does not have
`rustc`, `cargo`, or `/usr/bin/time`. Applying profile, dependency, linker, or
build-script changes without executable before/after evidence would be
speculative and would not satisfy the safety requirements for this work.

The branch is isolated. The pre-existing untracked `.vscode/mcp.json` was left
untouched.

## Baseline

| Metric | Before | After | Change |
| --- | ---: | ---: | ---: |
| Clean debug build | N/A | N/A | N/A |
| Incremental rebuild | N/A | N/A | N/A |
| Clean release build | N/A | N/A | N/A |
| Workspace tests | N/A | N/A | N/A |
| Peak compiler memory | N/A | N/A | N/A |
| Resolved dependencies | N/A | N/A | N/A |
| Duplicate dependencies | N/A | N/A | N/A |

See the files under `baseline/` for the exact command failures and static
inventory.

## Static optimization map

The checkout has 81 Cargo manifests and 18 build scripts. Two proc-macro crates
are present. The most relevant build-time operations are macOS bindgen/Metal/
Objective-C compilation in `warpui`, protobuf generation in `remote_server`,
schema registration in `graphql` and `warp_graphql_schema`, and Yarn-generated
command signatures.

The root profiles already contain explicit development tuning and comments
describing memory-sensitive release decisions. This makes a profile edit
particularly risky without measurements.

## Candidate workstreams

1. Measure `CARGO_BUILD_JOBS=1,2,4` for debug builds and record wall time and RSS.
2. Use `cargo tree -e features` and `cargo tree --duplicates` to identify a
   dependency or feature change with a measurable graph reduction.
3. Measure build-script invalidation before changing any `rerun-if-*` directives.
4. On macOS, measure `warpui` bindgen and native shader build invalidation.
5. Evaluate an alternative linker only on runners where it is already installed;
   keep any linker change in its own commit.

## Applied changes

None. There are no optimization commits to cherry-pick.

## Optional optimizations not applied

- Development profile changes: rejected pending compile-time and RSS evidence.
- Default-feature changes: rejected pending resolved graph and behavior tests.
- Build-script invalidation changes: deferred until platform-specific behavior
  and generated output dependencies are measured.
- Linker configuration: deferred because compatibility cannot be verified.
- Global job-count changes: deferred because the requested RSS measurements are
  unavailable.

## Required validation before continuing

Install the pinned Rust toolchain and `/usr/bin/time`, then run the baseline
commands from the request. Only candidates with measured benefit and passing
`cargo check`, relevant tests, and a clean rebuild should be committed.

## Final state

- Branch isolated: yes
- Atomic optimization commits: none created
- Tests: not run; Cargo unavailable
- Benchmarks recorded: command failures and static counts only
- Runtime behavior changed: no
- Unrelated files changed: no; pre-existing `.vscode/mcp.json` remains untouched
