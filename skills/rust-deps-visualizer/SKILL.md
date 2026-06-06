---
name: rust-deps-visualizer
description: "Visualize AND analyze Rust project dependencies — ASCII tree plus duplicate-version and feature-unification diagnosis. Triggers on: /deps-viz, dependency graph, show dependencies, visualize deps, duplicate dependencies, cargo tree duplicates, feature unification, why is this crate pulled in, 依赖图, 依赖可视化, 显示依赖, 重复依赖"
argument-hint: "[--depth N] [--features]"
allowed-tools: ["Bash", "Read", "Glob"]
---

# Rust Dependencies Visualizer

Generate ASCII art visualizations of your Rust project's dependency tree.

## Usage

```
/rust-deps-visualizer [--depth N] [--features]
```

**Options:**
- `--depth N`: Limit tree depth (default: 3)
- `--features`: Show feature flags

## Output Format

### Simple Tree (Default)

```
my-project v0.1.0
├── tokio v1.49.0
│   ├── pin-project-lite v0.2.x
│   └── bytes v1.x
├── serde v1.0.x
│   └── serde_derive v1.0.x
└── anyhow v1.x
```

### Feature-Aware Tree

```
my-project v0.1.0
├── tokio v1.49.0 [rt, rt-multi-thread, macros, fs, io-util]
│   ├── pin-project-lite v0.2.x
│   └── bytes v1.x
├── serde v1.0.x [derive]
│   └── serde_derive v1.0.x (proc-macro)
└── anyhow v1.x [std]
```

## Implementation

**Step 1:** Parse Cargo.toml for direct dependencies

```bash
cargo metadata --format-version=1 --no-deps 2>/dev/null
```

**Step 2:** Get full dependency tree

```bash
cargo tree --depth=${DEPTH:-3} ${FEATURES:+--features} 2>/dev/null
```

**Step 3:** Format as ASCII art tree

Use these box-drawing characters:
- `├──` for middle items
- `└──` for last items
- `│   ` for continuation lines

## Dependency Analysis (beyond the tree)

`cargo tree` draws the graph; the high-value use is **diagnosing** it. Three checks worth running on any non-trivial project:

**1. Duplicate versions** — the same crate compiled at two+ versions (slower builds, bigger binary, and confusing `expected X, found X` errors when a type leaks across the version boundary):

```bash
cargo tree -d            # --duplicates: lists every crate present at 2+ versions
```

**2. Who pulled it?** — invert the tree to find the dependency forcing an extra/old version:

```bash
cargo tree -i <crate>               # e.g. cargo tree -i hashbrown
cargo tree -i <crate> -e features   # ...and which feature path reaches it
```

**3. Feature unification** — Cargo unions features across the whole graph, so one dependency (even a dev- or build-dependency) enabling a feature turns it on for *everyone*. Trace it:

```bash
cargo tree -e features              # annotate every edge with the features it activates
cargo tree -e features -i <crate>   # why is <crate>'s feature X on? who switched it?
```

**Interpreting + fixing:**
- Duplicates *across a major version* (`1.x` vs `2.x`) are expected — they need an upstream bump, not your action.
- Duplicates *within a major* (`1.0.14` vs `1.0.21`) are usually fixable: unify the version in one place → see `m11-ecosystem` (the `[workspace.dependencies]` single-source-of-version pattern, and `[patch]` for a graph-wide override).
- Surprise features in a release build usually trace to a dependency's default features → pin `default-features = false` and re-add only what you need.

## Visual Enhancements

### Dependency Categories

```
my-project v0.1.0
│
├─[Runtime]─────────────────────
│ ├── tokio v1.49.0
│ └── async-trait v0.1.x
│
├─[Serialization]───────────────
│ ├── serde v1.0.x
│ └── serde_json v1.x
│
└─[Development]─────────────────
  ├── criterion v0.5.x
  └── proptest v1.x
```

### Size Visualization (Optional)

```
my-project v0.1.0
├── tokio v1.49.0        ████████████ 2.1 MB
├── serde v1.0.x         ███████ 1.2 MB
├── regex v1.x           █████ 890 KB
└── anyhow v1.x          ██ 120 KB
                         ─────────────────
                         Total: 4.3 MB
```

## Workflow

1. Check for Cargo.toml in current directory
2. Run `cargo tree` with specified options
3. Parse output and generate ASCII visualization
4. Optionally categorize by purpose (runtime, dev, build)

## Related Skills

| When | See |
|------|-----|
| Crate selection advice | m11-ecosystem |
| Workspace management | m11-ecosystem |
| Feature flag decisions | m11-ecosystem |
