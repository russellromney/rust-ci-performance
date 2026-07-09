# rust-ci-performance — deeper levers

Rarer / higher-effort levers. The playbook (`SKILL.md`) covers measure-first,
the triage, and the common levers. Open this when `cargo build --timings` says
you're **compile-bound on a fat crate**, or the common levers aren't enough.
Numbers marked **reported** are cited third-party claims — verify on your box.

## Structural — when compile-bound on a monolith

Only after `--timings` shows one/few crates dominating with low core
utilization. High ceiling, high effort.

### Split the giant crate (biggest structural win)
LLVM codegen is single-threaded per codegen unit and the *crate* is Cargo's unit
of parallelism — a monolith serializes the back-end no matter how many cores you
have. Splitting into workspace crates lets codegen run in parallel. Reported
extreme: Feldera's generated ~100k-line crate went 25–45 min → ~2 min by
splitting into ~1100 crates (saturating 128 threads).
- Footgun: a leaf crate depending on everything re-serializes the graph (the
  "diamond join"). Diminishing returns + per-crate fixed overhead.
- **Bumping `codegen-units` is cargo-culted** — no meaningful gain from 16→256 in
  practice. Parallelism comes from crate boundaries, not this knob.

### De-genericize hot generic code (guided by `cargo llvm-lines`)
Each generic instantiation is codegen'd separately — generic-heavy APIs explode
IR volume, the single-threaded slow thing. Refactor only what llvm-lines flags:
- Outer-generic / inner-concrete: thin generic wrapper converts to a concrete
  type, then calls a non-generic inner `fn` compiled once (stdlib pattern).
- `&dyn Trait` instead of generics on COLD paths: one codegen copy vs N (harmful
  on hot loops).
- `#[inline]` discipline: don't blanket it — forces cross-crate codegen dup.

### Nightly compiler levers (inconsistent, footgun-heavy)
Only if compile-bound and you can run nightly in CI:
- `RUSTFLAGS="-Z threads=8"` — parallel rustc front-end. Reported up to ~50% on
  front-end-bound code; barely moves codegen-bound monoliths (split those).
- Cranelift for DEBUG builds (`CARGO_PROFILE_DEV_CODEGEN_BACKEND=cranelift`) —
  faster, less-optimized codegen. **Hard-fails on inline-asm deps**; binaries
  unoptimized. Local iteration, not CI that also runs release.
- `-Z share-generics=y` — already on for dev/debug by default; off for release.

## Situational

- **cargo-hakari / workspace-hack** — a dep built with different feature sets by
  different workspace members compiles multiple times. hakari generates a
  `workspace-hack` crate unifying to the feature union so each builds once
  (reported up to ~1.7×). Only with real feature duplication — else pure
  overhead.
- **cargo-hack** — `cargo hack check --feature-powerset --depth 2` (NOT full
  powerset = 2^n) catches "feature X doesn't build alone" in one job instead of
  an exploding matrix.
- **cargo-chef** — only if CI builds a container image: caches dependency compile
  as a Docker layer (reported >50% when only source changes). **No-op on
  ephemeral CI unless the layer/BuildKit cache is persisted** (registry
  `--cache-to/--cache-from` or a cached-runner service).
- **Cached-runner services (Depot / Namespace / Blacksmith / RunsOn)** —
  persistent NVMe cache that makes `actions/cache` restore near-instant. Vendor
  numbers (~2×) — verify. **Fix caching before bigger raw runners**: 16-core vs
  8-core gave muted gains when cache variability dominated.

## Release-profile tuning (runtime perf, not build speed)

Off the CI-speed thesis, but frequently missing. A workspace with no
`[profile.release]` ships `lto=false, codegen-units=16` → zero cross-crate
inlining:
```toml
[profile.release]
lto = "thin"          # or "fat"
codegen-units = 1
# Do NOT blindly add panic = "abort" — see the SKILL.md gotcha.
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
