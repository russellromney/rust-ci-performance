---
name: rust-ci-performance
description: >
  Make a Rust project's CI and local test builds faster. Diagnose the real
  bottleneck (compile vs link vs test execution) with cargo build --timings,
  then apply the right lever: parallelize the CI gate, rust-cache/sccache,
  mold/lld linker, fewer test binaries, split a monolith crate, debuginfo
  reduction, cargo-nextest. Use when CI is slow, `cargo test`/`make verify`
  takes many minutes, a single sequential CI job dominates, or someone asks to
  speed up builds/tests/CI. MEASURE FIRST: do not guess which lever to pull.
metadata:
  author: russellromney
  version: "1.4"
  provenance: tina-rs CI speedup, 2026-07
---

# Rust CI / build performance playbook

Make Rust CI and test builds faster. Measure before you change anything.
Numbers below are **measured** on a real ~80k-LOC workspace (tina-rs: ~220 test
binaries, 3000+ tests, ubuntu + macOS). Numbers marked **reported** come from
someone else, so verify them yourself. Rarer, higher-effort levers live in
`REFERENCE.md`: splitting a monolith crate, de-genericizing, nightly rustc
flags, hakari, cargo-chef, cached runners.

## The one rule: MEASURE FIRST

Do not guess which lever to pull. The common mistake is to tune the cache when
the cache is already fine. On tina-rs we assumed the compile cache was broken.
We measured it. **sccache was at 85% hit rate.** The cache was fine. The slow
part was **linking 220 test binaries**, which no cache helps, plus debuginfo.
We fixed the linker and the test job dropped ~45%. Tuning the cache would have
done nothing.

### Diagnose the bottleneck (before any change)

A slow test job is one of three things:

1. **Compile-bound.** Most of the job is `Compiling ...` lines. Helped by
   caching, smaller crates, and less generic code.
2. **Link-bound.** Compiling finishes, but the job keeps running before tests
   start. Each test binary links on its own. Helped by a faster linker, fewer
   test binaries, and less debuginfo. Caching does not help linking.
3. **Execution-bound.** The tests themselves are slow. Only this case is helped
   by splitting tests across jobs.

Tools to tell them apart:
- **`cargo build --timings`** Start here. It writes an HTML chart of build time
  per crate. Shows if you are stuck compiling, stuck linking, or stuck on one
  big crate.
- **`sccache --show-stats`** Add it after the build step to print the cache hit
  rate. If the rate is high (over 70%) and the job is still slow, the cache is
  not your problem. Look at linking or test execution.
- **Timestamps in the log.** Compare the last `Compiling` line, the moment tests
  start, and the total job time. On tina-rs, 3013 tests ran in ~30s but the job
  took ~430s. Almost all of it was compile and link.

## The levers, in order of typical impact

### 1. Parallelize the CI gate into concurrent jobs (biggest immediate win)
One `make verify` job that runs every step in order costs the sum of all steps.
Split it into separate jobs that run at the same time. Now the total time is
just the slowest job. A good split: `static` (fmt+doc), `clippy`, `test`
(nextest+doctests), `guards` (loom/integration/cost). Measured: ~10–17 min down
to ~7–9.5 min. That is ~30–40% for free, before changing any compile settings.

Watch out for:
- **Weakening the gate by accident.** Branch protection lists required checks by
  job name. If you rename or split those jobs, the check can quietly stop being
  required. Check it with
  `gh api repos/OWNER/REPO/branches/BRANCH/protection` (404 means no
  protection). If protection exists, keep one umbrella job name or give the
  admin the new names.
- Every gating job must still run on PRs (`if: pull_request || push`).
- **Lints and docs must run on both operating systems.** clippy and `cargo doc`
  only check code that compiles for the current platform. A
  `cfg(target_os="macos")` block is only checked on macOS. `fmt` is the same
  everywhere, so ubuntu-only is fine for it.

### 2. Build cache: Swatinem/rust-cache (near-zero risk)
Caches `~/.cargo` and `target/`, keyed on `Cargo.lock` and the rustc version.
Big win on compiling dependencies. Almost always worth adding.
```yaml
- uses: Swatinem/rust-cache@v2
  with: { cache-on-failure: "true" }
```
- Set **`CARGO_INCREMENTAL: 0`** in CI. Incremental build files speed up repeat
  builds on your machine. In CI they just bloat `target/` and do not help, since
  each run starts fresh. Keep it on locally, off in CI.
- The cache key must include `Cargo.lock`, the toolchain, and the OS. Miss one
  and you load stale build files and trust them. rust-cache gets this right. A
  hand-written `actions/cache` step often gets it wrong.

### 3. sccache: only if the cache is actually the bottleneck
A cache that works per compiled unit, finer than rust-cache. Needs
`CARGO_INCREMENTAL=0`.
```yaml
env: { RUSTC_WRAPPER: sccache, CARGO_INCREMENTAL: "0", SCCACHE_GHA_ENABLED: "true" }
steps: [ ..., { uses: mozilla-actions/sccache-action@v0.0.10 } ]
```
rust-cache on its own is often enough. Check the hit rate (see Diagnose) before
you add sccache. If the rate is already high and the job is still slow, sccache
will not help. Look at linking instead.
- It does not cache linking. It can even make `--release` and LTO builds slower,
  because LTO output caches badly (one report: ~50% slower release). Use it for
  debug and test builds.
- Trap: setting `RUSTC_WRAPPER=sccache` at the workflow level leaks into jobs
  that do not install sccache, like a fuzz job, and breaks their builds. Turn it
  off per job with `env: { RUSTC_WRAPPER: "" }`.

### 4. Faster linker and fewer test binaries
Each test binary links on its own, and that can be most of the time. Two ways to
attack it.

**(a) Swap the linker.**
- **Linux: mold** (3–5× faster linking, reported). Install it
  (`rui314/setup-mold@v1` or `apt-get install -y mold`) and tell rustc to use it
  in `.cargo/config.toml`:
  ```toml
  [target.x86_64-unknown-linux-gnu]
  rustflags = ["-C", "link-arg=-fuse-ld=mold"]
  ```
  If mold chokes on a C dependency (like aws-lc-sys) or vendored code, fall back
  to `lld` (`-fuse-ld=lld`, ships with the toolchain, ~2–3×).
- **macOS: do not swap the linker.** Set `split-debuginfo = "unpacked"` in the
  dev/test profile. This skips the slow `dsymutil` step, which is most of the
  link time on macOS.

Measured: mold plus line-tables debuginfo cut the test job from **567s to 310s
on macOS (~45%) and 432s to 358s on ubuntu.**

**(b) Cut the number of test binaries.** Cargo builds and links one binary per
file in `tests/`. Merge many `tests/*.rs` files into one `tests/main.rs` that
pulls them in as `mod foo;`. Now you link once instead of N times (~50% reported
on a ~50-test suite where linking dominated). Cost: test-only helpers have to be
`pub`.

### 5. Reduce debuginfo (smaller objects, faster linking)
Tests rarely need full debug info. In the dev/test profile:
```toml
[profile.dev]
debug = "line-tables-only"   # keeps panic line numbers; much smaller
# or debug = 0 if you don't need line numbers in test panics
```

### 6. cargo-nextest (faster test runs, better isolation)
A faster test runner. It runs each test binary on its own, which fixes flaky
behavior you get when they share one `cargo test` run (for example trybuild
compile-fail suites clashing).
- **Gotcha: nextest does not run doctests.** Add a separate `cargo test --doc`,
  or you drop doctest coverage without noticing. Check the math: the
  `cargo test --workspace` count should equal the `nextest run` count plus the
  doctest count.
- **Only split tests across jobs when execution-bound.** Run
  `cargo nextest archive` once, then extract it on N jobs with
  `--partition count:i/N`. This builds once and never recompiles per shard. If
  you are compile- or link-bound (tests finish in seconds), the archive upload,
  download, and job startup cost more than you save. tina-rs skipped this on
  purpose.

## Cheap always-do wins

- **`concurrency: cancel-in-progress`** cancels old runs when you push again. It
  does not speed up a single run. It frees up runners and shortens the queue.
  **Footgun: do not use it on `main` or deploy workflows.** Limit it to PR
  branches.
  ```yaml
  concurrency: { group: "${{ github.workflow }}-${{ github.ref }}", cancel-in-progress: true }
  ```
- **Run cheap checks first** so a typo does not cost a 10-minute compile:
  `cargo fmt --check`, `typos`, `taplo fmt --check`, `cargo-machete`. Order them
  fast to slow: fmt, typos, taplo, then clippy and check, then build and test.
- **Remove unused dependencies** with `cargo-machete` or `cargo-shear`. Unused
  deps still compile and pull in their own proc-macro deps. Also set
  **`default-features = false`** on heavy crates like tokio or bindgen.
- **`cargo clippy` already does everything `cargo check` does.** If your CI runs
  both, drop the separate `check`. It is a full rebuild for nothing.

## Gotchas checklist (learned the hard way)

- **Three static gates.** A `make verify`-style CI runs fmt-check, clippy with
  `-D warnings`, and `cargo doc` with `RUSTDOCFLAGS="-D warnings"`. All three
  fail on warnings. Plain `cargo test` runs none of them. A public doc comment
  that links to a private item fails the doc gate. Run all three locally before
  you push.
- **Changing rustflags or the linker resets the cache.** The first run after you
  add mold rebuilds the cache and looks slow. Compare a warm run to another warm
  run, not to the rebuild run.
- **Do not add `panic = "abort"` for speed.** It breaks any crate that relies on
  `catch_unwind`, such as an actor runtime that turns a handler panic into an
  event. It is common advice, so check your code before you take it.
- **Do not fix a flaky timing test by raising its time limit.** The race is
  still there. Make it deterministic instead: gate the workers, assert on the
  mechanism, add a watchdog. `--retries` hides the flake, it does not fix it.

## TL;DR

1. Run `cargo build --timings` first. Everything else is a guess without it.
2. Split a sequential CI job into parallel jobs. Add rust-cache with
   `CARGO_INCREMENTAL=0`.
3. Check the hit rate and whether you are compile-, link-, or execution-bound.
4. Cache low: fix the cache. Cache fine but slow: linking (mold, split-debuginfo,
   fewer test binaries).
5. One big crate, or out of common levers: `REFERENCE.md`. Execution-bound:
   nextest archive and partition.
6. Always on PRs: cancel-in-progress, cheap checks first, three static gates.
