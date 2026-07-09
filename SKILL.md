---
name: rust-ci-performance
description: >
  Make a Rust project's CI and local test builds faster — diagnose the real
  bottleneck (compile vs link vs test execution) with cargo build --timings,
  then apply the right lever: parallelize the CI gate, rust-cache/sccache,
  mold/lld linker, fewer test binaries, split a monolith crate, debuginfo
  reduction, cargo-nextest. Use when CI is slow, `cargo test`/`make verify`
  takes many minutes, a single sequential CI job dominates, or someone asks to
  speed up builds/tests/CI. MEASURE FIRST — do not guess which lever to pull.
metadata:
  author: russellromney
  version: "1.3"
  provenance: tina-rs CI speedup, 2026-07
---

# Rust CI / build performance playbook

Cut Rust CI and test-build time, measure-first. Numbers below are **measured** on
a real ~80k-LOC workspace (tina-rs: ~220 test binaries, 3000+ tests, ubuntu +
macOS) unless marked **reported** (a cited third-party claim — verify it).
Rarer / higher-effort levers (splitting a monolith crate, de-genericizing,
nightly rustc flags, hakari, cargo-chef, cached runners) are in `REFERENCE.md`.

## The one rule: MEASURE FIRST

Do NOT guess which lever to pull. The most common mistake is "optimize the
cache" when the cache is already fine. On tina-rs the assumption was "the
compile cache isn't working"; instrumenting showed **sccache was at 85% hit
rate** — caching was not the problem. The real bottleneck was **linking 220
test binaries** (uncacheable) plus debuginfo. Fixing the linker cut the test
job ~45%; fixing the cache would have wasted the effort.

### Diagnose the bottleneck (before any change)

A slow `cargo test` / test CI job is one of three things:

1. **Compile-bound** — most job time is `Compiling ...` lines. Cache, fewer/
   smaller crates, less generic bloat help.
2. **Link-bound** — compile finishes but the job keeps going before tests
   start; many test binaries each link. Linker + fewer test binaries + less
   debuginfo help. Caching does NOT help linking.
3. **Execution-bound** — tests themselves are slow. Only here does test
   PARTITIONING help.

Tools to tell them apart:
- **`cargo build --timings`** — START HERE. HTML Gantt of per-crate build time +
  concurrency: compile-vs-link, and whether you're serialized on one fat crate.
- **`sccache --show-stats`** (add after the build step) — hit rate. High (>70%)
  + still-slow job ⇒ not a cache problem ⇒ link or execution.
- **Timestamps** — last `Compiling` line vs when tests start vs total job time.
  tina-rs: 3013 tests ran in ~30s but the job was ~430s ⇒ almost all
  compile+link ⇒ partitioning was correctly rejected.

## The levers, in order of typical impact

### 1. Parallelize the CI gate into concurrent jobs (biggest immediate win)
One sequential `make verify` job costs the SUM of its steps. Split into
independent jobs that run concurrently — wall-clock becomes the SLOWEST job. A
reasonable split: `static` (fmt+doc), `clippy`, `test` (nextest+doctests),
`guards` (loom/integration/cost). Measured: ~10–17 min → ~7–9.5 min, ~30–40%
for free before touching compilation.

CAVEATS:
- **Don't silently weaken the gate.** If branch protection lists a required
  check by job name, splitting/renaming jobs can disable it. Check
  `gh api repos/OWNER/REPO/branches/BRANCH/protection` (404 = no protection).
  If it exists, keep an umbrella job name or tell the admin the new names.
- Keep the PR trigger on every gating job (`if: pull_request || push`).
- **Platform-specific lints/docs must run on BOTH OSes.** clippy and `cargo doc`
  only lint code compiled for the active `cfg`; a `cfg(target_os="macos")` block
  is only linted on macOS. fmt is platform-independent (ubuntu-only is fine).

### 2. Build cache — Swatinem/rust-cache (near-zero risk)
Caches `~/.cargo` + `target/`, keyed on `Cargo.lock` + rustc. Huge win on
dependency compilation; almost always worth it.
```yaml
- uses: Swatinem/rust-cache@v2
  with: { cache-on-failure: "true" }
```
- Set **`CARGO_INCREMENTAL: 0`** in CI. Incremental artifacts speed repeated
  *local* builds but bloat `target/` and don't help fresh CI runs. On locally,
  off in CI.
- The key MUST include `Cargo.lock` + toolchain + OS or you reuse stale
  artifacts as valid. rust-cache does this; hand-rolled `actions/cache` often
  gets it wrong.

### 3. sccache — only if the cache is actually the bottleneck
Finer-grained per-compilation cache. Requires `CARGO_INCREMENTAL=0`.
```yaml
env: { RUSTC_WRAPPER: sccache, CARGO_INCREMENTAL: "0", SCCACHE_GHA_ENABLED: "true" }
steps: [ ..., { uses: mozilla-actions/sccache-action@v0.0.10 } ]
```
rust-cache alone is often enough. Check the hit rate (Diagnose) before adding
sccache; if it's already high and the job is still slow, sccache isn't your
lever — look at linking.
- **Does NOT cache linking, and can make `--release`/LTO builds SLOWER** (LTO
  artifacts cache poorly — one report: ~50% slower release). Use for debug/test.
- Trap: a workflow-level `RUSTC_WRAPPER=sccache` LEAKS into jobs that don't
  install sccache (e.g. a fuzz job) and fails their builds — override per-job
  with `env: { RUSTC_WRAPPER: "" }`.

### 4. Faster linker + fewer test binaries — the link bottleneck caching can't touch
Many test binaries each link separately, and that dominates. Two attacks:

**(a) Swap the linker.**
- **Linux — mold** (3–5× on link, reported). Install (`rui314/setup-mold@v1` or
  `apt-get install -y mold`) and point rustc at it via `.cargo/config.toml`:
  ```toml
  [target.x86_64-unknown-linux-gnu]
  rustflags = ["-C", "link-arg=-fuse-ld=mold"]
  ```
  Fallback: `lld` (`-fuse-ld=lld`, ships with the toolchain, ~2–3×) if mold
  chokes on a C dep (aws-lc-sys) or vendored code.
- **macOS — don't swap the linker; use `split-debuginfo = "unpacked"`** in the
  dev/test profile to skip the slow `dsymutil` step that dominates macOS link.

Measured: mold + line-tables debuginfo cut the test job **macOS 567s→310s
(~45%), ubuntu 432s→358s.**

**(b) Cut the NUMBER of test binaries.** Cargo compiles AND links one binary per
file in `tests/`. Consolidate many `tests/*.rs` into one `tests/main.rs` with
`mod foo;` submodules → one link step instead of N (~50% reported for a link-
dominated ~50-test suite). Cost: test-only internals must be `pub`.

### 5. Reduce debuginfo — smaller objects, faster linking
Tests rarely need full debuginfo. In the dev/test profile:
```toml
[profile.dev]
debug = "line-tables-only"   # keeps panic line numbers; much smaller
# or debug = 0 if you don't need line numbers in test panics
```

### 6. cargo-nextest — faster test EXECUTION + better isolation
Faster parallel runner; per-binary isolation fixes flaky aggregate-run behavior
(e.g. trybuild compile-fail suites colliding under one `cargo test`).
- **GOTCHA: nextest does NOT run doctests.** Add a separate `cargo test --doc`
  or you silently drop doctest coverage. Verify: `cargo test --workspace` total
  == `nextest run` count + doctest count.
- **Partition only when EXECUTION-bound.** `cargo nextest archive` once → extract
  on N shard jobs with `--partition count:i/N` (build once, never recompile per
  shard). If compile/link-bound (tests run in seconds), the archive
  upload/download + job startup exceeds the saving — tina-rs correctly skipped
  it.

## Cheap always-do wins

- **`concurrency: cancel-in-progress`** — stop wasting runners on superseded
  pushes. Zero per-run speedup, but reclaims capacity + shortens queues.
  **Footgun: NOT on `main`/deploy workflows** — scope to PR branches.
  ```yaml
  concurrency: { group: "${{ github.workflow }}-${{ github.ref }}", cancel-in-progress: true }
  ```
- **Pre-flight fail-fast checks** so a typo doesn't cost a 10-min compile:
  `cargo fmt --check`, `typos`, `taplo fmt --check`, `cargo-machete`. Order:
  fmt/typos/taplo → clippy/check → build/test.
- **Remove unused deps** (`cargo-machete`/`cargo-shear`) — dead deps still
  compile and drag their proc-macro deps. **`default-features = false`** on heavy
  deps (tokio, bindgen→clap).
- **`cargo clippy` is a superset of `cargo check`.** If a gate runs both, drop
  the standalone `check` — a wasted full workspace rebuild.

## Gotchas checklist (learned the hard way)

- **CI mistakes silently weaken the gate.** Preserve required-check job names,
  keep PR triggers, never let a "must pass" job become optional.
- **Three static gates.** `make verify`-style CI runs fmt-check, clippy
  `-D warnings`, AND `RUSTDOCFLAGS="-D warnings" cargo doc` — all fail on
  warnings; local `cargo test` runs none of them. A public doc comment linking a
  PRIVATE item fails the doc gate. Run all three locally before pushing.
- **Changing rustflags/linker invalidates the cache keys.** The first run after
  adding mold re-caches everything and looks slow. Compare WARM-vs-WARM (the
  second run), never the re-caching run.
- **Don't add `panic = "abort"` to chase perf** — it breaks any crate relying on
  `catch_unwind` (actor/supervision runtimes turning handler panics into
  events). Common advice; check before adopting.
- **Don't budget-widen flaky timing tests.** Deflake by determinism (gated
  workers, mechanism assertions, watchdogs) — widening a wall-clock bound leaves
  the race. `--retries` papers over, doesn't fix.

## TL;DR routing

1. `cargo build --timings` — guessing without this.
2. Sequential CI job → split into parallel jobs.
3. Add rust-cache + `CARGO_INCREMENTAL=0`.
4. Triage: hit rate + compile/link/execute.
5. Cache low → fix cache. Cache fine but slow → **linking**: mold + split-
   debuginfo + fewer test binaries.
6. Compile-bound on one fat crate, or common levers exhausted → `REFERENCE.md`.
7. Execution-bound (rare) → nextest archive + partition.
8. Always: cancel-in-progress (PR only), pre-flight checks, three static gates,
   warm-vs-warm measurement.
