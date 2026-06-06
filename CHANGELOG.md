# Changelog

## 1.1.0 — 2026-06-07

### Added
- **rust-deps-visualizer** — new "Dependency Analysis (beyond the tree)" section: duplicate-version detection (`cargo tree -d`), inverted who-pulls-this (`cargo tree -i [-e features]`), and feature-unification tracing (`cargo tree -e features`), each with interpretation and fixes that cross-reference `m11-ecosystem`'s `[workspace.dependencies]` / `[patch]` patterns. Updated the skill description/triggers to cover duplicate-dependency and feature-unification queries.

## 1.0.0 — 2026-06-07

Forked from [`actionbook/rust-skills`](https://github.com/actionbook/rust-skills) `2.0.9` / plugin `2.1.0` (MIT, ZhangHanDong) and enhanced with production-grade, source-cited Rust patterns.

### Added — 34 source-cited pattern enhancements across 18 skills

Each enhancement is a real pattern extracted from production Rust source, inserted additively at the exact gap it fills, with a `Company — file:line` citation. Source codebases: **LanceDB, DataFusion, Zed (gpui), Tauri, LakeSail, turbopuffer (tantivy/hnsw_rs), OpenAI Codex, Conductor**.

- **m01-ownership** — `PhantomData` variance marker table (Zed `gpui`)
- **m02-resource** — `Rc::new_cyclic` construction-order; fallible `Weak::upgrade()`
- **m03-mutability** — `mem::take`/`swap` to dodge borrow conflicts; `UnsafeCell` hot-path escape hatch
- **m04-zero-cost** — GAT + `dynamic()` escape hatch (static-by-default, erase on demand)
- **m05-type-driven** — `#[serde(try_from)]` validate-on-deserialize; exhaustive-match evolution safety
- **m06-error-handling** — workspace clippy `deny(unwrap/panic)`; `Arc<io::Error>` to derive `Clone`; `context!` macro with `file!()/line!()`
- **m07-concurrency** — RAII task-abort-on-`Drop`; actor own-state-no-lock pattern
- **m09-domain** — serde-boundary value-object validation
- **m10-performance** — mimalloc allocator swap; `target-cpu` RUSTFLAGS as SIMD precondition; cheap `O(1)` pre-filter
- **m11-ecosystem** — `[workspace.dependencies]` dedup; zero-dep base crate for compile cost
- **m12-lifecycle** — async RAII guard (`JoinHandle` abort-on-drop); `tokio::sync::OnceCell::get_or_try_init`
- **m13-domain-error** — `x-should-retry` server-driven retry; discount-only jitter vs thundering herd
- **m15-anti-pattern** — `mem::take` for move-out-of-`&mut self`; "best lock is no lock" actor alternative
- **domain-web** — `reqwest` pool/keep-alive knobs; honor `x-should-retry` on outbound
- **domain-cli** — orphan-free child spawn (two-guard discipline); `SharedChild` kill-while-reading
- **domain-cloud-native** — `tonic` gRPC service-to-actor adapter
- **domain-ml** — CPU-SIMD detect-once + mandatory scalar fallback; hand-written `Default` with per-param paper cites
- **domain-embedded** — `no_std` drops transcendental float math → `libm`

### Fixed — 5 correctness defects (caught by adversarial verification)

- **m01-ownership** — corrected the `PhantomData` variance table; the upstream reference had it wrong on 3/5 rows (`PhantomData<T>` is *covariant* not invariant; `fn(T) -> T` is *invariant* not covariant).
- **m07-concurrency** — corrected LakeSail actor field names; flagged illustrative type params as such.
- **m12-lifecycle** — removed two claims asserted beyond the cited source.
- **domain-cli** — separated verbatim source comments from an illustrative code skeleton (was presented as extracted).
- **rust-daily** — removed the nonexistent `agent-browser --limit` flag from 6 commands; top-N now happens in the parse step.

### Unchanged

All other upstream skills are preserved verbatim, including the 47-rule `unsafe-checker`, the LSP code-intelligence skills, and the 3-layer `rust-router`.
