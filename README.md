# Best Rust Patterns Skills

**Production-grade Rust skills for Claude Code.** A drop-in enhanced fork of [`actionbook/rust-skills`](https://github.com/actionbook/rust-skills) where every "correct but shallow" spot is backed by a **real pattern reverse-engineered from production Rust source** — LanceDB, DataFusion, Zed, Tauri, LakeSail, turbopuffer, OpenAI Codex, Conductor — each cited inline with `Company — file:line`.

38 skills · 34 source-cited enhancements across 18 skills · every enhancement adversarially verified against its cited source.

---

## Install

Pick whichever matches your setup — all three install the same 38 skills.

**[`npx skills`](https://github.com/vercel-labs/skills) (any agent):**
```bash
npx skills add LeventySeven/best-rust-patterns-skills
```

**Claude Code plugin marketplace:**
```
/plugin marketplace add LeventySeven/best-rust-patterns-skills
/plugin install best-rust-patterns@best-rust-patterns
```

**CoWork CLI:**
```bash
cowork install LeventySeven/best-rust-patterns-skills
```

> The 6 LSP code-intelligence skills (`rust-code-navigator`, `rust-call-graph`, `rust-trait-explorer`, `rust-symbol-analyzer`, `rust-refactor-helper`, plus call-hierarchy) need the `rust-analyzer-lsp` plugin and a `rust-analyzer` binary on `PATH`. Everything else works out of the box.

---

## What makes it "best patterns"

The upstream skills are accurate and well-structured — but, by design, thin: they route you to a concept, then stop. This fork keeps that routing and **fills the gaps with code that ships in real systems**:

| Skill | Enhancement | Source |
|-------|-------------|--------|
| `m01-ownership` | `PhantomData` variance marker table | Zed `gpui` |
| `m02-resource` | `Rc::new_cyclic`; fallible `Weak::upgrade()` | Zed · Codex |
| `m03-mutability` | `mem::take`/`swap`; `UnsafeCell` hot-path escape hatch | Zed · LanceDB |
| `m04-zero-cost` | GAT + `dynamic()` escape hatch | LanceDB |
| `m05-type-driven` | `#[serde(try_from)]` validation; exhaustive-match evolution | Codex · DataFusion |
| `m06-error-handling` | workspace `deny(unwrap)`; `Arc<io::Error>` for `Clone`; `context!` w/ `file!()` | LakeSail · turbopuffer · DataFusion |
| `m07-concurrency` | RAII task-abort-on-`Drop`; actor own-state-no-lock | Conductor · LakeSail |
| `m09-domain` | serde-boundary value-object validation | Codex |
| `m10-performance` | mimalloc swap; `target-cpu` flags; cheap `O(1)` pre-filter | LakeSail · turbopuffer · Zed |
| `m11-ecosystem` | `[workspace.dependencies]` dedup; zero-dep base crate | Zed · LakeSail |
| `m12-lifecycle` | async RAII guard; `OnceCell::get_or_try_init` | Conductor · LakeSail |
| `m13-domain-error` | `x-should-retry` server-driven retry; jitter vs thundering herd | turbopuffer |
| `m15-anti-pattern` | `mem::take` for move-out-of-`&mut`; "best lock is no lock" | Zed · LakeSail |
| `domain-web` | `reqwest` pool knobs; honor `x-should-retry` outbound | turbopuffer |
| `domain-cli` | orphan-free child spawn (two guards); `SharedChild` kill-while-reading | Codex · Tauri |
| `domain-cloud-native` | `tonic` gRPC service-to-actor adapter | LakeSail |
| `domain-ml` | CPU-SIMD detect-once + scalar fallback; hand-written `Default` w/ paper cites | LanceDB |
| `domain-embedded` | `no_std` kills float math → `libm` | turbopuffer |
| `rust-deps-visualizer` | duplicate-version + feature-unification diagnosis (`cargo tree -d`/`-i`/`-e features`) | cargo (native, v1.1.0) |

See [`CHANGELOG.md`](./CHANGELOG.md) for the full list and the 5 correctness fixes.

---

## The 38 skills

**Router** — `rust-router` triages every query into one of three layers and dispatches.

**Layer 1 · Language mechanics** — `m01-ownership` · `m02-resource` · `m03-mutability` · `m04-zero-cost` · `m05-type-driven` · `m06-error-handling` · `m07-concurrency`

**Layer 2 · Design choices** — `m09-domain` · `m10-performance` · `m11-ecosystem` · `m12-lifecycle` · `m13-domain-error` · `m14-mental-model` · `m15-anti-pattern`

**Layer 3 · Domain constraints** — `domain-web` · `domain-cli` · `domain-cloud-native` · `domain-embedded` · `domain-fintech` · `domain-iot` · `domain-ml`

**Cross-cutting** — `coding-guidelines` (50+ style rules + clippy map) · `unsafe-checker` (47 soundness rules + checklists)

**LSP code intelligence** — `rust-code-navigator` · `rust-call-graph` · `rust-trait-explorer` · `rust-symbol-analyzer` · `rust-refactor-helper` · `rust-deps-visualizer`

**Utility** — `rust-daily` (Rust news) · `rust-learner` (crate/version/docs lookup) · `rust-skill-creator` · `meta-cognition-parallel`

**Support** — `core-actionbook` · `core-agent-browser` · `core-dynamic-skills` · `core-fix-skill-docs`

---

## Provenance & quality

- **Source-cited.** Every added pattern carries a `Company — teardown:line` citation back to the production code it came from. No generic blog advice.
- **Adversarially verified.** Each enhancement was independently re-checked against its cited source for faithfulness, additivity, and tightness before shipping. That pass caught and fixed 5 defects — including a variance error inherited from the reference material.
- **Additive only.** Enhancements are new sections/rows; no upstream content was rewritten or deleted.

---

## Credits

- Upstream: **[`actionbook/rust-skills`](https://github.com/actionbook/rust-skills)** by **ZhangHanDong** — MIT.
- `unsafe-checker` rules derive from **[rust-coding-guidelines-zh](https://github.com/Rust-Coding-Guidelines/rust-coding-guidelines-zh)**.
- Enhancement patterns are extracted from the public source of the projects cited inline, for educational reference.

## License

MIT — see [`LICENSE`](./LICENSE). Derivative of `actionbook/rust-skills` (MIT), with enhancements © 2026 LeventySeven.
