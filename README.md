# rust-ci-performance

A measure-first agent **skill** for making a Rust project's CI and local test
builds faster. It diagnoses the *real* bottleneck — compile vs link vs test
execution — and applies the right lever instead of guessing.

Works with Claude Code, Cursor, Windsurf, Copilot, Codex, Aider, Zed, Amp,
Cline, OpenCode, Kimi, Grok, pi, and pretty much any other agent that supports
skills.

Every number in the skill is **measured**, not estimated — from cutting a real
~80k-LOC Rust workspace's CI (~220 test binaries, 3000+ tests) on Linux + macOS
runners.

## Install

```sh
npx skills add russellromney/rust-ci-performance
```

That's it. The CLI figures out which agents you have and installs the skill to
the right place. (It reads this public repo directly — nothing is published to
the npm registry.) Add `--global` for a user-level install, or `--all` to
install to every detected agent without prompts.

> The `skills` CLI was formerly `add-skill`; `npx add-skill …` still works but
> prints a deprecation notice — prefer `npx skills add`.

After installing, just ask your agent:

> speed up my Rust CI — it takes 15 minutes

> my `cargo test` job is slow, what's the bottleneck?

`CLAUDE.md` and `AGENTS.md` are symlinks to `SKILL.md`, so the same content
works across agent conventions.

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

## Manual install

If `add-skill` doesn't work for your setup, install manually. Each command drops
`SKILL.md` where that agent looks for skills. Use `~/…` for a user-global install
or the project-relative path for one repo.

<details>
<summary><b>Claude Code</b></summary>

```sh
# user-global
git clone https://github.com/russellromney/rust-ci-performance.git ~/.claude/skills/rust-ci-performance
# or per-project
git clone https://github.com/russellromney/rust-ci-performance.git .claude/skills/rust-ci-performance
```
</details>

<details>
<summary><b>OpenAI Codex</b></summary>

```sh
git clone https://github.com/russellromney/rust-ci-performance.git ~/.codex/skills/rust-ci-performance
# or per-project: .codex/skills/rust-ci-performance
```
</details>

<details>
<summary><b>Grok</b></summary>

```sh
git clone https://github.com/russellromney/rust-ci-performance.git ~/.grok/skills/rust-ci-performance
```
</details>

<details>
<summary><b>Kimi (kimi-code)</b></summary>

```sh
git clone https://github.com/russellromney/rust-ci-performance.git ~/.kimi-code/skills/rust-ci-performance
```
</details>

<details>
<summary><b>OpenCode</b></summary>

```sh
git clone https://github.com/russellromney/rust-ci-performance.git .opencode/skills/rust-ci-performance
# or user-global: ~/.config/opencode/skill/rust-ci-performance
```
</details>

<details>
<summary><b>Cursor</b></summary>

```sh
git clone https://github.com/russellromney/rust-ci-performance.git .cursor/skills/rust-ci-performance
# or drop the rule text directly:
curl -o .cursorrules https://raw.githubusercontent.com/russellromney/rust-ci-performance/main/SKILL.md
```
</details>

<details>
<summary><b>Windsurf</b></summary>

```sh
mkdir -p .windsurf/rules
curl -o .windsurf/rules/rust-ci-performance.md https://raw.githubusercontent.com/russellromney/rust-ci-performance/main/SKILL.md
```
</details>

<details>
<summary><b>Cline / Roo Code</b></summary>

```sh
mkdir -p .clinerules
curl -o .clinerules/rust-ci-performance.md https://raw.githubusercontent.com/russellromney/rust-ci-performance/main/SKILL.md
```
</details>

<details>
<summary><b>Aider</b></summary>

```sh
git clone https://github.com/russellromney/rust-ci-performance.git
aider --read rust-ci-performance/SKILL.md
```
</details>

<details>
<summary><b>pi</b></summary>

```sh
git clone https://github.com/russellromney/rust-ci-performance.git ~/.pi/skills/rust-ci-performance
```
</details>

Any other agent: point it at `SKILL.md` however it loads skills or rules — the
whole skill is that one file.

## License

MIT © Russell Romney
