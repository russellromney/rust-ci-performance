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
  version: "1.1"
  provenance: >
    tina-rs CI speedup, 2026-07 (numbers marked "measured" are from that work;
    numbers marked "reported" are cited third-party claims — verify on your box)
---

# Rust CI / build performance playbook

Evidence-based playbook for cutting Rust CI and test-build time. Numbers marked
**measured** were measured on a real ~80k-LOC workspace (tina-rs, ~220 test
binaries / 3000+ tests, ubuntu + macOS runners). Numbers marked **reported** are
third-party claims with a source — directional, not gospel; measure on your own
project before trusting them.

## The one rule that matters: MEASURE FIRST

Do NOT guess which lever to pull. The most common mistake is "optimize the
cache" when the cache is already fine. On tina-rs the assumption was "the
compile cache isn't working"; instrumenting showed **sccache was at 85% hit
rate** (measured) — caching was not the problem at all. The real bottleneck was
**linking 220 test binaries** (uncacheable) plus debuginfo. Fixing the cache
would have wasted effort; fixing the linker cut the test job ~45% (measured).

### Diagnose the bottleneck (do this before any change)

A slow `cargo test` / test CI job is one of three things. Find out which:

1. **Compile-bound** — most job time is `Compiling ...` lines. Cache, fewer/
   smaller crates, less generic bloat help.
2. **Link-bound** — compile finishes but the job keeps going before tests
   start; many separate test binaries each link. Linker + fewer test binaries +
   less debuginfo help. Caching does NOT help linking.
3. **Execution-bound** — tests themselves are slow (each takes seconds, or
   long-running integration tests). Only here does test PARTITIONING help.

### Diagnostic tools (reach for these, don't eyeball)

- **`cargo build --timings`** — START HERE. Emits an HTML Gantt of per-crate
  build time + concurrency. Tells you compile-vs-link and whether you're
  serialized on one fat crate or genuinely parallel. Every lever below is a
  guess until you've read this.
- **`cargo llvm-lines`** — ranks functions by LLVM IR generated and monomorph
  copy count. The map for generic-bloat work (which crate/fn to de-genericize).
- **`cargo bloat --time`** — per-crate build-time attribution; complements
  `--timings`.
- **`RUSTFLAGS="-Z time-passes"`** (nightly) — where rustc spends time. This is
  how Feldera found 86% of a build was LLVM passes + codegen (reported). Use
  when `--timings` fingers one crate and you need to know *why* it's slow.

Quick triage from CI logs without extra tooling:
- Add `sccache --show-stats` after the build and read the hit rate. High
  (>70%) + still-slow job ⇒ NOT a cache problem ⇒ it's link or execution.
- Compare the last `Compiling` line's timestamp vs when tests start vs total
  job time. On tina-rs: 3013 tests ran in ~30s but the job was ~430s ⇒ almost
  all compile+link ⇒ **partitioning test execution was correctly rejected**.

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
~30–40% for free, before touching compilation at all.

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
Under `--locked` the key is exact. One step per build job:
```yaml
- uses: Swatinem/rust-cache@v2
  with: { cache-on-failure: "true" }
```
Huge win on dependency compilation (identical run-to-run). Almost always worth
it. Scope weekly/fuzz jobs with `workspaces:`.
- Set **`CARGO_INCREMENTAL: 0`** in CI env. Incremental artifacts speed
  *repeated local* builds but bloat `target/` and don't help fresh CI runs that
  have no prior incremental state. Leave incremental ON locally, OFF in CI.
- Cache-key hygiene: the key MUST include `Cargo.lock` + toolchain + OS or you
  reuse stale artifacts as valid (mystery rebuilds / link errors). rust-cache
  does this; hand-rolled `actions/cache` often gets it wrong.
- GHA evicts caches unused for 7 days (LRU). The old 10 GB/repo cap was lifted
  2025-11-20, so `target/` size is less of a budget problem — but `actions/
  cache` still saves/restores `target/` as one coarse blob (slow, includes
  junk), which is the durable reason to prefer sccache's fetch-what-you-need.

### 3. sccache — only if the cache is actually the bottleneck
Finer-grained per-compilation cache. Requires `CARGO_INCREMENTAL=0` (can't cache
incremental artifacts).
```yaml
env: { RUSTC_WRAPPER: sccache, CARGO_INCREMENTAL: "0", SCCACHE_GHA_ENABLED: "true" }
steps: [ ..., { uses: mozilla-actions/sccache-action@v0.0.10 } ]
```
**Measure its hit rate before assuming it helps.** For a single repo, rust-cache
ALONE is often enough. If the hit rate is already high (we saw 85%, measured)
and the job is still slow, sccache is NOT your lever — stop tuning it and look
at linking.
- Backends: GHA (easy, shares GHA cache mechanics) or S3/Redis/WebDAV for a
  shared cross-runner cache that bypasses GHA limits.
- **sccache does NOT cache linking, and can make `--release`/LTO builds SLOWER**
  (LTO artifacts cache poorly — one report saw ~50% slower release). Use it for
  debug/test compilation, not blindly on release.
- Trap: a workflow-level `RUSTC_WRAPPER=sccache` env LEAKS into jobs that don't
  install sccache (e.g. a fuzz job) and fails their builds — override per-job
  with `env: { RUSTC_WRAPPER: "" }`.

### 4. Faster linker + fewer test binaries — the link bottleneck caching can't touch
When there are many test binaries, each links separately and that dominates.
Two independent attacks:

**(a) Swap the linker.**
- **Linux — mold** (3–5× on link, reported). Install (`rui314/setup-mold@v1` or
  `apt-get install -y mold`) and point rustc at it via `.cargo/config.toml`:
  ```toml
  [target.x86_64-unknown-linux-gnu]
  rustflags = ["-C", "link-arg=-fuse-ld=mold"]
  ```
  Fallback: `lld` (`-fuse-ld=lld`, ships with the toolchain, ~2–3×) if mold
  chokes on a C dep (aws-lc-sys) or vendored code.
- **macOS — don't swap the linker (fiddly); use `split-debuginfo = "unpacked"`**
  in the dev/test profile to skip the slow `dsymutil` step that dominates macOS
  link time.
Measured on tina-rs: mold + line-tables debuginfo cut the test job
**macOS 567s→310s (~45%), ubuntu 432s→358s.** The real lever after the cache
turned out fine.

**(b) Cut the NUMBER of test binaries.** Cargo compiles AND links one binary per
file in `tests/`. Consolidate many `tests/*.rs` into a single `tests/main.rs`
with `mod foo;` submodules → one link step instead of N. (~50% reported for a
link-dominated ~50-test suite.) Complements the linker win — fewer binaries to
compile too. Cost: test-only internals must be `pub`. `tests/common/` helper
subdirs already avoid extra binaries.

### 5. Reduce debuginfo — smaller objects, faster linking
Full debuginfo is a big chunk of both compile and link time. Tests rarely need
it. In the dev/test profile:
```toml
[profile.dev]
debug = "line-tables-only"   # keeps panic/backtrace line numbers; much smaller
# or debug = 0 if you don't need line numbers in test panics
```
Also shrinks `target/` (smaller cache). Complements the linker win.

### 6. cargo-nextest — faster test EXECUTION + better isolation
Faster parallel runner; per-binary isolation fixes flaky aggregate-run behavior
(e.g. trybuild compile-fail suites colliding under one `cargo test`).
- **GOTCHA: nextest does NOT run doctests.** You MUST add a separate
  `cargo test --doc` run or you silently drop doctest coverage. Verify:
  `cargo test --workspace` total == `nextest run` count + doctest count.
- Install fast via `taiki-e/install-action@nextest`.
- **Partition only when EXECUTION-bound.** `cargo nextest archive` once →
  extract on N shard jobs with `--partition count:i/N` (build once, never
  recompile per shard — the whole point). If the job is compile/link-bound
  (tests run in seconds), the archive upload/download + job startup exceeds the
  saving. tina-rs was compile/link-bound → correctly NOT partitioned.
- Useful knobs: `--no-fail-fast`/`--max-fail=N` (full failure list per CI run);
  setup-scripts (start a DB once, not per test); test-groups (serialize tests
  contending on a shared resource instead of dropping global `--test-threads`).

### 7. Release-profile tuning (runtime perf, not build speed — but often missing)
Not a CI-speed lever, but frequently absent and worth adding for perf-sensitive
crates. A workspace with no `[profile.release]` ships `lto=false,
codegen-units=16` and gets zero cross-crate inlining:
```toml
[profile.release]
lto = "thin"          # or "fat"
codegen-units = 1
# Do NOT blindly add panic = "abort" — it breaks any crate relying on
# catch_unwind (e.g. actor/supervision runtimes that turn handler panics into
# events). Check before adding.
```

## Structural levers — when compile-bound on a monolith

Only reach here after `cargo build --timings` shows one/few crates dominating
with low core utilization. High ceiling, high effort.

### Split the giant crate (the biggest structural win)
LLVM codegen is single-threaded per codegen unit and the *crate* is Cargo's unit
of parallelism. A monolithic crate serializes the expensive back-end no matter
how many cores you have. Splitting into workspace crates lets codegen run in
parallel. Reported extreme: Feldera's generated ~100k-line crate went 25–45 min
→ ~2 min by splitting into ~1100 crates (saturating 128 threads).
- Footgun: a leaf crate that depends on everything re-serializes the graph (the
  "diamond join"). Diminishing returns + per-crate fixed overhead.
- **`codegen-units` bumping is cargo-culted** — Feldera saw no meaningful gain
  from 16→256. Real parallelism comes from crate boundaries, not this knob.

### De-genericize hot generic code (guided by `cargo llvm-lines`)
Each generic instantiation is codegen'd separately (per type, per crate without
share-generics). Generic-heavy APIs explode IR volume — the single-threaded slow
thing.
- Outer-generic / inner-concrete: thin generic wrapper converts to a concrete
  type, then calls a non-generic inner `fn` compiled once (stdlib pattern).
- `&dyn Trait` instead of generics on COLD paths: one codegen copy vs N. Harmful
  on hot loops.
- `#[inline]` discipline: don't blanket it — forces cross-crate codegen dup.
Refactor only what llvm-lines actually flags; the indirection has a cost.

### Nightly compiler levers (inconsistent, footgun-heavy)
Try only if compile-bound and you can run nightly in CI:
- `RUSTFLAGS="-Z threads=8"` — parallel rustc front-end (parse/typeck/borrowck).
  Reported up to ~50%, ~30% avg on front-end-bound code; barely moves codegen-
  bound monoliths (fix those with crate splitting instead).
- Cranelift backend for DEBUG builds (`CARGO_PROFILE_DEV_CODEGEN_BACKEND=
  cranelift`) — faster, less-optimized codegen. **Hard-fails on inline-asm
  deps**; binaries unoptimized. Best for local iteration, dubious for CI that
  also runs release.
- `-Z share-generics=y` — already on for dev/debug by default; off for release
  (hurts codegen). Explicit control is nightly and has caused ICEs.

## Cheap always-do wins

- **`concurrency: cancel-in-progress`** — stop wasting runners on superseded
  pushes:
  ```yaml
  concurrency:
    group: ${{ github.workflow }}-${{ github.ref }}
    cancel-in-progress: true
  ```
  Zero per-run speedup but reclaims capacity + shortens queues (= real latency).
  **Footgun: do NOT set on `main`/deploy workflows** where every commit must
  finish — scope to PR branches.
- **Pre-flight fail-fast checks before the expensive compile.** Run cheap gates
  first so a typo doesn't cost a 10-min compile: `cargo fmt --check`, `typos`
  (crate-ci/typos, sub-second), `taplo fmt --check` (TOML), `cargo-machete`
  (dead deps). Order: fmt/typos/taplo → clippy/check → build/test. Fail cheap,
  fail early.
- **Remove unused deps** — `cargo-machete`/`cargo-shear` (fast, pure-source,
  some false positives) or `cargo-udeps` (accurate, needs nightly + full build).
  Dead deps still compile and drag their proc-macro deps.
- **`default-features = false`** on heavy deps + explicit features. Many crates
  pull heavy optional code by default (tokio, bindgen→clap). Free once found.
- **`cargo clippy` is a superset of `cargo check`.** If a gate runs both, drop
  the standalone `check` — it's a wasted full workspace rebuild.

## Situational levers

- **cargo-hakari / workspace-hack** — in a workspace, a dep built with different
  feature sets by different members compiles multiple times. hakari generates a
  `workspace-hack` crate unifying to the feature union so each dep builds once
  (reported up to ~1.7×). Useless on small workspaces or when members already
  share features (pure overhead + a generated crate to keep current). Only if
  you actually have feature duplication.
- **cargo-hack for feature testing** — `cargo hack check --feature-powerset
  --depth 2` (NOT full powerset = 2^n) catches "feature X doesn't build alone"
  in one job instead of an exploding matrix. A correctness lever that *saves*
  matrix jobs.
- **cargo-chef (only if CI builds a container image)** — splits dependency
  compilation into a cacheable Docker layer; source-only changes skip straight
  to the final build (reported >50%). **No-op on ephemeral CI unless the layer/
  BuildKit cache is persisted** (registry `--cache-to/--cache-from` or a cached-
  runner service). Combine with sccache — different granularity.
- **Cached-runner services (Depot / Namespace / Blacksmith / RunsOn)** — fast
  persistent cache (mounted NVMe) that makes `actions/cache` restore near-
  instant, often the real bottleneck. Vendor numbers (~2×, ~half cost) —
  verify. Skeptic note: **fix caching before reaching for bigger raw runners** —
  16-core vs 8-core gave muted overall gains when cache variability dominated
  (reported); cache effectiveness beats core count.

## Gotchas checklist (learned the hard way)

- **CI mistakes silently weaken the gate.** Preserve required-check job names,
  keep PR triggers, never let a "must pass" job become optional.
- **The three static gates.** `make verify`-style CI runs fmt-check, clippy
  `-D warnings`, AND `RUSTDOCFLAGS="-D warnings" cargo doc` — all three fail on
  warnings. Local `cargo test` runs NONE of them. A doc comment linking a
  PUBLIC item to a PRIVATE item fails the doc gate. Run all three locally
  before pushing.
- **Changing rustflags/linker invalidates rust-cache + sccache keys.** The
  FIRST run after adding mold re-caches everything and looks slow. Compare
  WARM-vs-WARM (the second run), never the re-caching run.
- **Don't run `cargo doc` in the PR gate unless you're checking it** (broken
  intra-doc links / build). It's a full codegen-adjacent pass; if you publish
  docs, do it in a separate non-blocking job with its own cache.
- **Don't budget-widen flaky timing tests.** Deflake by determinism (gated
  workers, mechanism assertions, watchdog deadlock detectors) — widening a
  wall-clock bound leaves the race. nextest `--retries` papers over, doesn't fix.
- **PR-branch caches aren't fully shared with the default branch's first run**
  (GHA cache scoping). The true warm number shows on the SECOND run of a branch.

## Quick measurement recipe

```sh
# where is the time going, per crate (open the HTML it prints)
cargo build --timings
# per-job durations of a run
gh run view <run-id> --json jobs -q '.jobs[] | "\(.name): \((( (.completedAt|fromdateiso8601)-(.startedAt|fromdateiso8601)) ))s"'
# sccache hit rate (add `- run: sccache --show-stats` after the build first)
gh run view <run-id> --log | grep -iE "Compile requests|Cache hits|Cache misses|hit rate"
# overall wall-clock
gh run view <run-id> --json createdAt,updatedAt -q '(((.updatedAt|fromdateiso8601)-(.createdAt|fromdateiso8601)))'
```

## TL;DR decision tree

1. **Measure first**: `cargo build --timings`, then `-Z time-passes` if one
   crate dominates. Everything below is guessing without this.
2. Single sequential CI job? → split into parallel jobs (~30–40%, free).
3. Add `Swatinem/rust-cache` + `CARGO_INCREMENTAL=0` in CI (near-zero risk).
4. Instrument: sccache `--show-stats` + compile/link/execute triage.
5. **Cache hit-rate low** → fix cache (or add sccache, but not on release/LTO).
   **Cache fine but job slow** → it's LINKING: mold (Linux) + split-debuginfo
   (macOS) + line-tables debuginfo + fewer test binaries. Usually the real win.
6. **Compile-bound on a monolith** (timings shows one fat crate) → split the
   crate; de-genericize via llvm-lines. Don't bump codegen-units.
7. **Execution-bound** (rare) → nextest archive + partition. Otherwise skip
   partitioning.
8. Always cheap: cancel-in-progress (PR branches only), pre-flight fail-fast
   checks, remove dead deps, three static gates locally, warm-vs-warm
   measurement, don't weaken the gate.
