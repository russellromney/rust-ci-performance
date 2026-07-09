---
name: rust-ci-performance
description: >
  Make a Rust project's CI and local test builds faster — diagnose the real
  bottleneck (compile vs link vs test execution) and apply the right lever:
  parallelize the CI gate, rust-cache/sccache, mold/lld linker, debuginfo
  reduction, cargo-nextest. Use when CI is slow, `cargo test`/`make verify`
  takes many minutes, a single sequential CI job dominates, or someone asks to
  speed up builds/tests/CI. MEASURE FIRST — do not guess which lever to pull.
metadata:
  author: Russell Romney
  version: "1.0"
  provenance: distilled from a real ~80k-LOC Rust workspace CI speedup; all numbers below are measured, not estimated
---

# Rust CI / build performance playbook

Evidence-based playbook for cutting Rust CI and test-build time. Every number
here was measured on a real ~80k-LOC Rust workspace with ~220 test
binaries / 3000+ tests, ubuntu + macOS runners.

## The one rule that matters: MEASURE FIRST

Do NOT guess which lever to pull. The most common mistake is "optimize the
cache" when the cache is already fine. In one real case the assumption was "the
compile cache isn't working"; instrumenting showed **sccache was at 85% hit
rate** — caching was not the problem at all. The real bottleneck was **linking
220 test binaries** (uncacheable) plus debuginfo generation. Fixing the cache
would have wasted effort; fixing the linker cut the test job ~45%.

### Diagnose the bottleneck (do this before any change)

A slow `cargo test` / test CI job is one of three things. Find out which:

1. **Compile-bound** — most job time is `Compiling ...` lines. Cache + fewer
   codegen units help.
2. **Link-bound** — compile finishes but the job keeps going before tests
   start; many separate test binaries each link. Linker + debuginfo help.
   Caching does NOT help linking.
3. **Execution-bound** — tests themselves are slow (each takes seconds, or
   there are long-running integration tests). Only here does test PARTITIONING
   help.

How to tell them apart:
- Add `sccache --show-stats` after the build step and read the hit rate from
  the CI log. High hit rate (>70%) + still-slow job ⇒ NOT a cache problem ⇒
  it's link or execution.
- Compare the timestamp of the last `Compiling` line vs when tests start vs
  total job time. If tests run in ~30s but the job is ~400s, you are
  compile/link-bound, and **partitioning test execution is near-useless**.
- Real example: 3013 tests executed in ~30s; the job was ~430s. Almost all of it
  was compile+link. Partitioning was correctly rejected.

## The levers, in order of typical impact

### 1. Parallelize the CI gate into concurrent jobs (biggest immediate win)
If CI runs one sequential `make verify` / `cargo test`+clippy+doc job, its time
is the SUM of all steps. Split into independent jobs that run concurrently —
wall-clock becomes the SLOWEST job, not the sum. A reasonable split:
- `static` — fmt + doc
- `clippy` — clippy (its own job)
- `test` — nextest + doctests
- `guards` — loom / integration-guard / cost checks
Measured: ~10–17 min sequential → ~7–9.5 min (slowest job = the test job).
That is ~30–40% for free, before touching compilation at all.

CAVEATS:
- **Do not silently weaken the gate.** If branch protection lists a required
  check by job name, renaming/splitting jobs can disable the gate. Check
  `gh api repos/OWNER/REPO/branches/BRANCH/protection` (404 = no protection =
  safe to rename). If protection exists, keep an umbrella job name or tell the
  admin the new required-check names.
- Keep the PR trigger on every gating job (`if: pull_request || push`).
- **Platform-specific lints/docs must run on BOTH OSes.** clippy and `cargo
  doc` only lint code compiled for the active `cfg`; a `cfg(target_os="macos")`
  block is only linted when clippy/doc runs on macOS. fmt is platform-
  independent (ubuntu-only is fine).

### 2. Build cache — Swatinem/rust-cache (near-zero risk)
Caches `~/.cargo` (registry/git) + `target/`, keyed on `Cargo.lock` + rustc.
Under `--locked` the key is exact. Add one step per build job:
```yaml
- uses: Swatinem/rust-cache@v2
  with: { cache-on-failure: "true" }
```
Huge win on dependency compilation (which is identical run-to-run). This is
almost always worth it. For the weekly/fuzz jobs, scope with `workspaces:`.

### 3. sccache — only if the cache is actually the bottleneck
Finer-grained per-compilation cache (GHA backend). Requires
`CARGO_INCREMENTAL=0` (sccache can't cache incremental artifacts).
```yaml
env: { RUSTC_WRAPPER: sccache, CARGO_INCREMENTAL: "0", SCCACHE_GHA_ENABLED: "true" }
steps: [ ..., { uses: mozilla-actions/sccache-action@v0.0.10 } ]
```
**Measure its hit rate before assuming it helps.** For a single repo, rust-cache
ALONE is often enough and sccache adds a moving part. If the hit rate is already
high (we saw 85%) and the job is still slow, sccache is NOT your lever — stop
tuning it and look at linking. Trap: a workflow-level `RUSTC_WRAPPER=sccache`
env LEAKS into jobs that don't install sccache (e.g. a fuzz job) and fails their
builds — override per-job with `env: { RUSTC_WRAPPER: "" }`.

### 4. Faster linker — attacks the link bottleneck caching can't touch
When there are many test binaries, each links separately and that dominates.
- **Linux — mold** (3–5× on link). Install (`rui314/setup-mold@v1` or
  `apt-get install -y mold`) and point rustc at it via `.cargo/config.toml`
  (target-scoped, cleaner than env):
  ```toml
  [target.x86_64-unknown-linux-gnu]
  rustflags = ["-C", "link-arg=-fuse-ld=mold"]
  ```
  Fallback: `lld` (`-fuse-ld=lld`, ships with the toolchain, ~2–3×) if mold
  chokes on a C dep (aws-lc-sys) or vendored code.
- **macOS — don't swap the linker (fiddly); use `split-debuginfo = "unpacked"`**
  in the dev/test profile to skip the slow `dsymutil` step that dominates macOS
  link time.
Measured: mold + line-tables debuginfo cut the test job
**macOS 567s→310s (~45%), ubuntu 432s→358s.** This was the real lever after
the cache turned out to be fine.

### 5. Reduce debuginfo — smaller objects, faster linking
Full debuginfo is a big chunk of both compile and link time. Tests rarely need
it. In the dev/test profile:
```toml
[profile.dev]
debug = "line-tables-only"   # keeps panic/backtrace line numbers; much smaller
# or debug = 0 if you don't need line numbers in test panics
```
Complements the linker win (smaller objects link faster).

### 6. cargo-nextest — faster test EXECUTION + better isolation
Faster parallel test runner; per-binary isolation fixes flaky aggregate-run
behavior (e.g. trybuild compile-fail suites colliding under one `cargo test`).
- **GOTCHA: nextest does NOT run doctests.** You MUST add a separate
  `cargo test --doc` run or you silently drop doctest coverage. Verify counts:
  `cargo test --workspace` total == `nextest run` count + doctest count.
- Install fast via `taiki-e/install-action@nextest`.
- **Archive-based partitioning** (`cargo nextest archive` once → N shard jobs
  `--partition count:i/N`) only pays when EXECUTION-bound. If the job is
  compile/link-bound (tests run in seconds), partitioning's artifact
  upload/download + job-startup overhead exceeds what it saves — skip it. We
  measured tina-rs as compile/link-bound and correctly did NOT partition.

### 7. Release-profile tuning (runtime perf, not build speed — but often missing)
Not a CI-speed lever, but frequently absent and worth adding for perf-sensitive
crates. A workspace with no `[profile.release]` ships `lto=false,
codegen-units=16` and gets zero cross-crate inlining:
```toml
[profile.release]
lto = "thin"          # or "fat"
codegen-units = 1
# Do NOT blindly add panic = "abort" — it breaks any crate that relies on
# catch_unwind (e.g. actor/supervision runtimes that turn handler panics into
# events). Check before adding.
```

## Gotchas checklist (learned the hard way)

- **CI mistakes silently weaken the gate.** Preserve required-check job names,
  keep PR triggers, never let a job that "must pass" become optional.
- **The three static gates.** `make verify`-style CI runs fmt-check, clippy
  `-D warnings`, AND `RUSTDOCFLAGS="-D warnings" cargo doc` — all three fail on
  warnings. Local `cargo test` runs NONE of them. A doc comment linking a
  PUBLIC item to a PRIVATE item fails the doc gate. Run all three locally
  before pushing.
- **Changing rustflags/linker invalidates rust-cache + sccache keys.** The
  FIRST run after adding mold re-caches everything and looks slow. Compare
  WARM-vs-WARM (the second run), never the re-caching run.
- **`cargo clippy` is a superset of `cargo check`.** If a gate runs both, drop
  the standalone `check` — it's a wasted full workspace rebuild.
- **Don't budget-widen flaky timing tests.** Deflake by determinism (gated
  workers, mechanism assertions, watchdog deadlock detectors) — widening a
  wall-clock bound leaves the race. nextest `--retries` can paper over flakes
  but doesn't fix them.
- **PR-branch caches aren't fully shared with the default branch's first run**
  (GHA cache scoping). The true warm number shows on the SECOND run of a branch.

## Quick measurement recipe

```sh
# per-job durations of a run
gh run view <run-id> --json jobs -q '.jobs[] | "\(.name): \((( (.completedAt|fromdateiso8601)-(.startedAt|fromdateiso8601)) ))s"'
# sccache hit rate (add `- run: sccache --show-stats` after the build first)
gh run view <run-id> --log | grep -iE "Compile requests|Cache hits|Cache misses|hit rate"
# overall wall-clock
gh run view <run-id> --json createdAt,updatedAt -q '(((.updatedAt|fromdateiso8601)-(.createdAt|fromdateiso8601)))'
```

## TL;DR decision tree

1. Single sequential CI job? → split into parallel jobs first (~30–40%, free).
2. Add `Swatinem/rust-cache` (near-zero risk).
3. Instrument: sccache `--show-stats` + compile/link/execute triage.
4. Cache hit-rate low → fix cache (or add sccache). Cache fine but job slow →
   it's LINKING: add mold (Linux) + split-debuginfo (macOS) + line-tables
   debuginfo. This is usually the real win.
5. Tests execution-bound (rare)? → nextest archive + partition. Otherwise skip
   partitioning.
6. Always: three static gates locally, warm-vs-warm measurement, don't weaken
   the gate.
