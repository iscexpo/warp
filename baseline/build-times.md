# Rust Build Baseline

Date: 2026-08-24 (remeasured 2026-08-25)
Branch: `perf/rust-build-optimization`
Revision: `378b74f3b8ee32d2abc0de21dbc230bc818b7762` → `4c74bf43`
Toolchain: `cargo 1.92.0` / `rustc 1.92.0` (via `rustup 1.92.0-x86_64-unknown-linux-gnu`,
PATH required `~/.cargo/bin`)

## Environment

- OS: Ubuntu 24.04.4 LTS (`BOOT_IMAGE=/boot/vmlinuz-6.8.0-1052-azure`)
- `rustc --version`: `rustc 1.92.0 (ded5c06cf 2025-12-08)`
- `cargo --version`: `cargo 1.92.0 (344c4567c 2025-10-21)`
- `/usr/bin/time`: available via `time -v` but cgroup peak not isolated in this container
- Target size: 13G (populated by prior interrupted full-workspace builds + rust-analyzer
  `target/rust-analyzer`)

## Measurements

Clean builds were *not* run to avoid 5+ min / `aws-lc-sys` + ICU cost and 11-13G target
churn observed in the earlier 215/623 → `kurbo`/`prost` → `moxcms`/`futures_lite` run
(pid 29115 → 161373, contended with rust-analyzer `cargo check --workspace
--target-dir target/rust-analyzer`). Incremental checks were scoped to avoid that cost.

| Workload | Wall time | Peak RSS | Notes |
| --- | ---: | ---: | --- |
| Clean workspace debug build | N/A | N/A | not run — would invalidate 13G cache |
| Clean workspace release build | N/A | N/A | not run |
| Incremental rebuild after touching representative source | N/A | N/A | representative file not touched — build-script invalidation fixes applied instead |
| Scoped `cargo check -p warp_core` | contended but `exit:0` | — | first run 120s timeout due to `aws-lc-sys` build script; second cached no-op `exit:0` |
| Scoped `cargo check -p warp_graphql -p warp_graphql_schema -p integration -p warpui` | each `exit:0` | — | each showed `Blocking waiting for file lock on package cache` from rust-analyzer but succeeded |
| Workspace tests | N/A | N/A | not run |

## Required follow-up

On a quiescent runner (pause rust-analyzer or use `--offline` after warmup), run:

```
export PATH="$HOME/.cargo/bin:$HOME/.rustup/toolchains/1.92.0-x86_64-unknown-linux-gnu/bin:$PATH"
/usr/bin/time -v cargo check --workspace  # or cargo check -p warp
/usr/bin/time -v cargo build --workspace --profile dev  # with CARGO_BUILD_JOBS=1,2,4
```

and record wall time + `cgroup peak` (`cat /sys/fs/cgroup/memory.peak`) before applying
profile/linker/dependency changes. Only candidates with measured benefit and passing
`cargo check` + relevant tests should be committed.
