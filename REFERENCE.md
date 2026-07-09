# rust-ci-performance: deeper levers

Rarer, higher-effort levers. The playbook (`SKILL.md`) covers measure-first, the
triage, and the common levers. Open this when `cargo build --timings` shows you
are **compile-bound on one big crate**, or when the common levers are not
enough. Numbers marked **reported** come from someone else, so verify them
yourself.

## Structural levers: when compile-bound on one big crate

Use these only after `--timings` shows one or a few crates taking most of the
time while your cores sit idle. Big payoff, big effort.

### Split the giant crate (biggest structural win)
LLVM compiles one codegen unit at a time, and Cargo runs one crate per job slot.
So one giant crate runs the slow back-end on a single core, no matter how many
cores you have. Split it into several workspace crates and they compile in
parallel. Reported case: Feldera's generated ~100k-line crate went from 25–45
min to ~2 min after they split it into ~1100 crates and used all 128 threads.
- Footgun: if one crate depends on all the others, everything waits on it again
  (the "diamond join"). Returns shrink as you split, and each crate adds fixed
  overhead.
- **Raising `codegen-units` is a myth.** Going from 16 to 256 gave no real gain
  in practice. The parallelism comes from splitting crates, not this setting.

### De-genericize hot generic code (guided by `cargo llvm-lines`)
Every use of a generic with a new type compiles a fresh copy. Generic-heavy code
makes a huge amount of IR, and compiling IR is the slow single-threaded step.
Only change what `cargo llvm-lines` flags:
- Keep a thin generic wrapper that converts its argument to a concrete type,
  then calls a plain inner function that compiles once. The standard library
  does this a lot.
- On cold paths, take `&dyn Trait` instead of a generic. That compiles one copy
  instead of many. Do not do it on hot loops.
- Do not put `#[inline]` on everything. It makes other crates recompile the
  function too.

### Nightly compiler levers (inconsistent, footgun-heavy)
Only worth it if you are compile-bound and can run nightly in CI:
- `RUSTFLAGS="-Z threads=8"` runs the rustc front end in parallel. Reported up
  to ~50% faster on front-end-bound code. It barely helps a codegen-bound
  monolith; split that instead.
- Cranelift for debug builds (`CARGO_PROFILE_DEV_CODEGEN_BACKEND=cranelift`)
  compiles faster but optimizes less. It fails outright on deps that use inline
  asm, and the binaries are slow. Good for local iteration, not for CI that also
  builds release.
- `-Z share-generics=y` is already on for dev and debug builds. It is off for
  release.

## Situational levers

- **cargo-hakari / workspace-hack.** When workspace members turn on different
  features for the same dependency, that dependency compiles more than once.
  hakari writes a `workspace-hack` crate that turns on the full set of features
  once, so each dependency builds a single time (reported up to ~1.7×). Only
  helps if you have this duplication. Otherwise it is pure overhead.
- **cargo-hack.** `cargo hack check --feature-powerset --depth 2` checks feature
  combinations up to two at a time. It catches "feature X does not build on its
  own" in one job. Skip the full powerset; it is 2^n combinations.
- **cargo-chef.** Only if CI builds a container image. It caches the dependency
  build as a Docker layer, so a source-only change skips it (reported over 50%
  faster). It does nothing on throwaway CI runners unless you save the layer
  cache, either with a registry (`--cache-to`/`--cache-from`) or a cached-runner
  service.
- **Cached-runner services (Depot, Namespace, Blacksmith, RunsOn).** They keep a
  fast cache on local disk, so restoring `actions/cache` is almost instant.
  Their numbers (~2×) come from vendors, so verify. Fix caching before you pay
  for bigger runners. Going from 8 to 16 cores gave small gains once cache speed
  was the real limit.

## Release-profile tuning (runtime speed, not build speed)

This is about runtime speed, not build speed, but it is often missing. A
workspace with no `[profile.release]` uses `lto=false` and `codegen-units=16`,
so it never inlines across crates:
```toml
[profile.release]
lto = "thin"          # or "fat"
codegen-units = 1
# Do not add panic = "abort" blindly. See the panic gotcha in SKILL.md.
```

## Extended measurement

```sh
# per-job durations of a run
gh run view <run-id> --json jobs -q '.jobs[] | "\(.name): \((( (.completedAt|fromdateiso8601)-(.startedAt|fromdateiso8601)) ))s"'
# overall wall-clock
gh run view <run-id> --json createdAt,updatedAt -q '(((.updatedAt|fromdateiso8601)-(.createdAt|fromdateiso8601)))'
# per-crate timing HTML
cargo build --timings
# deep rustc profile (nightly): front-end vs LLVM codegen
RUSTFLAGS="-Z time-passes" cargo build
```
