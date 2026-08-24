# Rust Build Baseline

Date: 2026-08-24
Branch: `perf/rust-build-optimization`
Revision: `378b74f3b8ee32d2abc0de21dbc230bc818b7762`

## Environment

- OS: Ubuntu 24.04.4 LTS
- `rustc --version`: unavailable (`rustc: command not found`)
- `cargo --version`: unavailable (`cargo: command not found`)
- `/usr/bin/time`: unavailable

## Measurements

The requested commands could not execute because this dev container has no Rust
toolchain or `/usr/bin/time` installed. No timing or memory values are inferred.

| Workload | Wall time | Peak RSS |
| --- | ---: | ---: |
| Clean workspace debug build | N/A | N/A |
| Clean workspace release build | N/A | N/A |
| Incremental rebuild after touching a representative source file | N/A | N/A |
| Workspace tests | N/A | N/A |

The representative source file was not touched because no build could validate
the resulting incremental behavior.

## Required follow-up

Install the repository's pinned toolchain, install `/usr/bin/time`, and rerun the
commands from the optimization document before applying or evaluating changes.
