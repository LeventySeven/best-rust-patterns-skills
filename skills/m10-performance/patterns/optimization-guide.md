# Rust Performance Optimization Guide

## Profiling First

### Tools
```bash
# CPU profiling
cargo install flamegraph
cargo flamegraph --bin myapp

# Memory profiling
cargo install cargo-instruments  # macOS
heaptrack ./target/release/myapp  # Linux

# Benchmarking
cargo bench  # with criterion

# Cache analysis
valgrind --tool=cachegrind ./target/release/myapp
```

### Criterion Benchmarks
```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn benchmark_parse(c: &mut Criterion) {
    let input = "test data".repeat(1000);

    c.bench_function("parse_v1", |b| {
        b.iter(|| parse_v1(&input))
    });

    c.bench_function("parse_v2", |b| {
        b.iter(|| parse_v2(&input))
    });
}

criterion_group!(benches, benchmark_parse);
criterion_main!(benches);
```

---

## Algorithm-Level: Cheap Pre-Filter Before the Expensive Path

The biggest wins (10-1000x) come from doing less work, not faster work. A recurring move: compute a cheap, *conservative* fingerprint for each candidate and the query, reject non-matches with a single comparison, and only run the expensive comparison (DP, full scan, distance calc) on survivors. The filter must never produce false negatives.

Zed's fuzzy matcher packs a string's character set into a `u64` `CharBag` and gates every candidate with one AND + one compare before any dynamic-programming scoring — discarding ~99% of candidates (Zed — `crates/fuzzy/src/char_bag.rs`):

```rust
pub struct CharBag(u64);

impl CharBag {
    pub fn is_superset(self, other: CharBag) -> bool {
        self.0 & other.0 == other.0
    }
}
```

`is_superset` is `self.0 & other.0 == other.0`: if the candidate's set doesn't contain every query character, it cannot match, skip it. Generalize the shape: a Bloom-filter membership check before a DB hit, a bounding-box overlap test before per-point geometry, a min/max range check before a row scan.

- The filter must be *conservative* (over-accepts, never wrongly rejects) — correctness lives in the expensive path, speed lives in the cheap gate.
- Profile the accept rate: the win comes from a low survivor fraction. If the filter passes most candidates it's pure overhead.

---

## Common Optimizations

### 1. Avoid Unnecessary Allocations

```rust
// BAD: allocates on every call
fn to_uppercase(s: &str) -> String {
    s.to_uppercase()
}

// GOOD: return Cow, allocate only if needed
use std::borrow::Cow;

fn to_uppercase(s: &str) -> Cow<'_, str> {
    if s.chars().all(|c| c.is_uppercase()) {
        Cow::Borrowed(s)
    } else {
        Cow::Owned(s.to_uppercase())
    }
}
```

### 2. Reuse Allocations

```rust
// BAD: creates new Vec each iteration
for item in items {
    let mut buffer = Vec::new();
    process(&mut buffer, item);
}

// GOOD: reuse buffer
let mut buffer = Vec::new();
for item in items {
    buffer.clear();
    process(&mut buffer, item);
}
```

### 3. Use Appropriate Collections

| Need | Collection | Notes |
|------|------------|-------|
| Sequential access | `Vec<T>` | Best cache locality |
| Random access by key | `HashMap<K, V>` | O(1) lookup |
| Ordered keys | `BTreeMap<K, V>` | O(log n) lookup |
| Small sets (<20) | `Vec<T>` + linear search | Lower overhead |
| FIFO queue | `VecDeque<T>` | O(1) push/pop both ends |

### 4. Pre-allocate Capacity

```rust
// BAD: many reallocations
let mut v = Vec::new();
for i in 0..10000 {
    v.push(i);
}

// GOOD: single allocation
let mut v = Vec::with_capacity(10000);
for i in 0..10000 {
    v.push(i);
}
```

---

## String Optimization

### Avoid String Concatenation in Loops

```rust
// BAD: O(n²) allocations
let mut result = String::new();
for s in strings {
    result = result + &s;
}

// GOOD: O(n) with push_str
let mut result = String::new();
for s in strings {
    result.push_str(&s);
}

// BETTER: pre-calculate capacity
let total_len: usize = strings.iter().map(|s| s.len()).sum();
let mut result = String::with_capacity(total_len);
for s in strings {
    result.push_str(&s);
}

// BEST: use join for simple cases
let result = strings.join("");
```

### Use &str When Possible

```rust
// BAD: requires allocation
fn greet(name: String) {
    println!("Hello, {}", name);
}

// GOOD: borrows, no allocation
fn greet(name: &str) {
    println!("Hello, {}", name);
}

// Works with both:
greet("world");                    // &str
greet(&String::from("world"));     // &String coerces to &str
```

---

## Iterator Optimization

### Use Iterators Over Indexing

```rust
// BAD: bounds checking on each access
let mut sum = 0;
for i in 0..vec.len() {
    sum += vec[i];
}

// GOOD: no bounds checking
let sum: i32 = vec.iter().sum();

// GOOD: when index needed
for (i, item) in vec.iter().enumerate() {
    // ...
}
```

### Lazy Evaluation

```rust
// Iterators are lazy - computation happens at collect
let result: Vec<_> = data
    .iter()
    .filter(|x| x.is_valid())
    .map(|x| x.process())
    .take(10)  // stop after 10 items
    .collect();
```

### Avoid Collecting When Not Needed

```rust
// BAD: unnecessary intermediate allocation
let filtered: Vec<_> = items.iter().filter(|x| x.valid).collect();
let count = filtered.len();

// GOOD: no allocation
let count = items.iter().filter(|x| x.valid).count();
```

---

## Parallelism with Rayon

```rust
use rayon::prelude::*;

// Sequential
let sum: i32 = (0..1_000_000).map(|x| x * x).sum();

// Parallel (automatic work stealing)
let sum: i32 = (0..1_000_000).into_par_iter().map(|x| x * x).sum();

// Parallel with custom chunk size
let results: Vec<_> = data
    .par_chunks(1000)
    .map(|chunk| process_chunk(chunk))
    .collect();
```

---

## Memory Layout

### Use Appropriate Integer Sizes

```rust
// If values are small, use smaller types
struct Item {
    count: u8,      // 0-255, not u64
    flags: u8,      // small enum
    id: u32,        // if 4 billion is enough
}
```

### Pack Structs Efficiently

```rust
// BAD: 24 bytes due to padding
struct Bad {
    a: u8,   // 1 byte + 7 padding
    b: u64,  // 8 bytes
    c: u8,   // 1 byte + 7 padding
}

// GOOD: 16 bytes (or use #[repr(packed)])
struct Good {
    b: u64,  // 8 bytes
    a: u8,   // 1 byte
    c: u8,   // 1 byte + 6 padding
}
```

### Box Large Values

```rust
// Large enum variants waste space
enum Message {
    Quit,
    Data([u8; 10000]),  // all variants are 10000+ bytes
}

// Better: box the large variant
enum Message {
    Quit,
    Data(Box<[u8; 10000]>),  // variants are pointer-sized
}
```

---

## Async Performance

### Avoid Blocking in Async

```rust
// BAD: blocks the executor
async fn bad() {
    std::thread::sleep(Duration::from_secs(1));  // blocking!
    std::fs::read_to_string("file.txt").unwrap();  // blocking!
}

// GOOD: use async versions
async fn good() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    tokio::fs::read_to_string("file.txt").await.unwrap();
}

// For CPU work: spawn_blocking
async fn compute() -> i32 {
    tokio::task::spawn_blocking(|| {
        heavy_computation()
    }).await.unwrap()
}
```

### Buffer Async I/O

```rust
use tokio::io::{AsyncBufReadExt, BufReader};

// BAD: many small reads
async fn bad(file: File) {
    let mut byte = [0u8];
    while file.read(&mut byte).await.unwrap() > 0 {
        process(byte[0]);
    }
}

// GOOD: buffered reading
async fn good(file: File) {
    let reader = BufReader::new(file);
    let mut lines = reader.lines();
    while let Some(line) = lines.next_line().await.unwrap() {
        process(&line);
    }
}
```

---

## Release Build Optimization

### Cargo.toml Settings

```toml
[profile.release]
lto = true           # Link-time optimization
codegen-units = 1    # Single codegen unit (slower compile, faster code)
panic = "abort"      # Smaller binary, no unwinding
strip = true         # Strip symbols

[profile.release-fast]
inherits = "release"
opt-level = 3        # Maximum optimization

[profile.release-small]
inherits = "release"
opt-level = "s"      # Optimize for size
```

### Swap the Global Allocator

For allocation-heavy binaries (query engines, ML inference, anything churning many heap buffers), replacing the system allocator is one of the cheapest wins — typically 10-30% faster with zero code changes beyond one declaration. Production example (LakeSail's `sail-cli` — `crates/sail-cli/src/main.rs`):

```rust
#[cfg(feature = "mimalloc")]
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

- Gate it behind a `#[cfg(feature = "mimalloc")]` so it can be turned off (e.g. for valgrind/ASan compatibility).
- Rust allows exactly ONE `#[global_allocator]` — make sure a dependency isn't pulling in a second (e.g. `jemallocator`) transitively.
- Biggest gains show up with large/frequent allocations (Arrow buffers, parse arenas); for tiny short-lived programs the difference is noise.

### Target the Actual CPU (unlock SIMD)

The SIMD/parallelism tier (2-8x) is mostly invisible unless you tell LLVM which instructions it may emit. By default `cargo build --release` targets a conservative baseline (roughly `x86-64-v2` / SSE4.2) — AVX2/AVX-512 and most auto-vectorization never fire. Production example (turbopuffer's Makefile):

```makefile
target/release/%: src/bin/%.rs
	@RUSTFLAGS='-C target-cpu=native' cargo build --release --bin $(notdir $@)
```

- `target-cpu=native` = every instruction the build host has. Use it when you control the deploy hardware (your own cloud VMs / on-prem).
- A `native`-built binary **crashes on older CPUs** lacking the assumed instructions. For binaries you ship to unknown machines, use `-C target-cpu=x86-64-v3` (Haswell baseline, includes AVX2) instead.
- Pass via `RUSTFLAGS=...` or pin per-target in `.cargo/config.toml` under `[build] rustflags`.

### Compile-Time Assertions

```rust
// Zero runtime cost
const _: () = assert!(std::mem::size_of::<MyStruct>() <= 64);
```

---

## Checklist

Before optimizing:
- [ ] Profile to find actual bottlenecks
- [ ] Have benchmarks to measure improvement
- [ ] Consider if optimization is worth complexity

Common wins:
- [ ] Reduce allocations (Cow, reuse buffers)
- [ ] Use appropriate collections
- [ ] Pre-allocate with_capacity
- [ ] Use iterators instead of indexing
- [ ] Enable LTO for release builds
- [ ] Use rayon for parallel workloads
