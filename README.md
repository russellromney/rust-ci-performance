# rust-ci-performance

An agent **skill** that makes a Rust project's CI and local test builds faster —
the measure-first way. It diagnoses the *real* bottleneck (compile vs link vs
test execution) and applies the right lever instead of guessing.

Every number in the skill is **measured**, not estimated — from cutting a real
~80k-LOC Rust workspace's CI (~220 test binaries, 3000+ tests) on Linux + macOS
runners.

## Why

The most common Rust-CI mistake is "optimize the cache" when the cache is
already fine. On the workspace this skill came from, the assumption was "the
compile cache isn't working" — instrumenting showed **sccache was at 85% hit
rate**. Caching was never the problem. The real bottleneck was **linking 220
test binaries** (uncacheable) plus debuginfo. Fixing the linker (mold +
line-tables debuginfo) cut the test job **~45%**; fixing the cache would have
wasted the effort.

So the skill's first rule is **MEASURE FIRST**, and it gives you the triage to
tell compile-bound from link-bound from execution-bound before you touch
anything.

## What's in it

- The measure-first rule + a compile/link/execute triage recipe
- The levers in order of typical impact:
  1. Parallelize the CI gate into concurrent jobs (~30–40%, free)
  2. `Swatinem/rust-cache` (near-zero risk)
  3. `sccache` — only if the cache is actually the bottleneck
  4. Faster linker (mold on Linux, `split-debuginfo` on macOS) — the lever
     caching can't touch
  5. Debuginfo reduction
  6. `cargo-nextest` (+ the doctest gotcha)
  7. Release-profile tuning (with the `panic = "abort"` trap called out)
- A gotchas checklist learned the hard way (silently weakening the gate, the
  three static gates, warm-vs-warm measurement, rustflags invalidating caches)
- Copy-paste `gh` measurement commands

## Install (local)

Not published to an skills registry yet — install by copying `SKILL.md` into your
agent's skills directory.

```sh
git clone https://github.com/russellromney/rust-ci-performance.git

# Claude Code
mkdir -p ~/.claude/skills/rust-ci-performance
cp rust-ci-performance/SKILL.md ~/.claude/skills/rust-ci-performance/

# Codex / Grok (same layout)
cp rust-ci-performance/SKILL.md ~/.codex/skills/rust-ci-performance/SKILL.md
cp rust-ci-performance/SKILL.md ~/.grok/skills/rust-ci-performance/SKILL.md

# OpenCode
mkdir -p ~/.config/opencode/skill/rust-ci-performance
cp rust-ci-performance/SKILL.md ~/.config/opencode/skill/rust-ci-performance/SKILL.md
```

`AGENTS.md` points agents at `SKILL.md`, mirroring the convention used by other
agent-skill repos.

## Use

Once installed, ask your agent to speed up CI and it will apply the playbook:

> speed up my Rust CI — it takes 15 minutes

> my `cargo test` job is slow, what's the bottleneck?

## License

MIT © Russell Romney
