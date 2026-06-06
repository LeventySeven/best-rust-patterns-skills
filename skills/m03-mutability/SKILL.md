---
name: m03-mutability
description: "CRITICAL: Use for mutability issues. Triggers: E0596, E0499, E0502, cannot borrow as mutable, already borrowed as immutable, mut, &mut, interior mutability, Cell, RefCell, Mutex, RwLock, 可变性, 内部可变性, 借用冲突"
user-invocable: false
---

# Mutability

> **Layer 1: Language Mechanics**

## Core Question

**Why does this data need to change, and who can change it?**

Before adding interior mutability, understand:
- Is mutation essential or accidental complexity?
- Who should control mutation?
- Is the mutation pattern safe?

---

## Error → Design Question

| Error | Don't Just Say | Ask Instead |
|-------|----------------|-------------|
| E0596 | "Add mut" | Should this really be mutable? |
| E0499 | "Split borrows" | Is the data structure right? |
| E0502 | "Separate scopes" | Why do we need both borrows? |
| RefCell panic | "Use try_borrow" | Is runtime check appropriate? |

---

## Thinking Prompt

Before adding mutability:

1. **Is mutation necessary?**
   - Maybe transform → return new value
   - Maybe builder → construct immutably

2. **Who controls mutation?**
   - External caller → `&mut T`
   - Internal logic → interior mutability
   - Concurrent access → synchronized mutability

3. **What's the thread context?**
   - Single-thread → Cell/RefCell
   - Multi-thread → Mutex/RwLock/Atomic

---

## Trace Up ↑

When mutability conflicts persist:

```
E0499/E0502 (borrow conflicts)
    ↑ Ask: Is the data structure designed correctly?
    ↑ Check: m09-domain (should data be split?)
    ↑ Check: m07-concurrency (is async involved?)
```

| Persistent Error | Trace To | Question |
|-----------------|----------|----------|
| Repeated borrow conflicts | m09-domain | Should data be restructured? |
| RefCell in async | m07-concurrency | Is Send/Sync needed? |
| Mutex deadlocks | m07-concurrency | Is the lock design right? |

---

## Trace Down ↓

From design to implementation:

```
"Need mutable access from &self"
    ↓ T: Copy → Cell<T>
    ↓ T: !Copy → RefCell<T>

"Need thread-safe mutation"
    ↓ Simple counters → AtomicXxx
    ↓ Complex data → Mutex<T> or RwLock<T>

"Need shared mutable state"
    ↓ Single-thread: Rc<RefCell<T>>
    ↓ Multi-thread: Arc<Mutex<T>>
```

---

## Borrow Rules

```
At any time, you can have EITHER:
├─ Multiple &T (immutable borrows)
└─ OR one &mut T (mutable borrow)

Never both simultaneously.
```

## Quick Reference

| Pattern | Thread-Safe | Runtime Cost | Use When |
|---------|-------------|--------------|----------|
| `&mut T` | N/A | Zero | Exclusive mutable access |
| `Cell<T>` | No | Zero | Copy types, no refs needed |
| `RefCell<T>` | No | Runtime check | Non-Copy, need runtime borrow |
| `Mutex<T>` | Yes | Lock contention | Thread-safe mutation |
| `RwLock<T>` | Yes | Lock contention | Many readers, few writers |
| `Atomic*` | Yes | Minimal | Simple types (bool, usize) |

### When `RefCell`'s runtime check is the bottleneck

`Cell`/`RefCell` are safe wrappers over `UnsafeCell`. If profiling shows the per-access borrow-flag check in a hot loop is the cost, AND you can prove access is sequential (single consumer, no aliasing), drop to `UnsafeCell` directly:

```rust
pub struct PostingIterator {
    compressed: Option<UnsafeCell<CompressedState>>,
}
// access skips RefCell's borrow-check branch:
fn compressed_state_ptr(&self) -> *mut CompressedState {
    self.compressed.as_ref().unwrap().get()   // *mut, no runtime guard
}
```

**Only after measuring.** This removes the runtime guard the `RefCell` row above charges you for — and removes the safety net with it. You now owe a `// SAFETY:` proof that no two live `&mut` overlap (see `unsafe-checker`). Default to `RefCell`.

*Source: LanceDB `lance-index/src/scalar/inverted/wand.rs` — WAND inner loop.*

## Error Code Reference

| Error | Cause | Quick Fix |
|-------|-------|-----------|
| E0596 | Borrowing immutable as mutable | Add `mut` or redesign |
| E0499 | Multiple mutable borrows | Restructure code flow |
| E0502 | &mut while & exists | Separate borrow scopes |

---

## Interior Mutability Decision

| Scenario | Choose |
|----------|--------|
| T: Copy, single-thread | `Cell<T>` |
| T: !Copy, single-thread | `RefCell<T>` |
| T: Copy, multi-thread | `AtomicXxx` |
| T: !Copy, multi-thread | `Mutex<T>` or `RwLock<T>` |
| Read-heavy, multi-thread | `RwLock<T>` |
| Simple flags/counters | `AtomicBool`, `AtomicUsize` |

---

## Mutate by Replacement (no `&mut` conflict, no clone)

When you must read-and-replace a field (the E0499/E0502 case where you'd otherwise hold two borrows), swap ownership instead of borrowing twice:

```rust
// Take: replace field with T::default(), return the old value. One op, no clone.
let comments = std::mem::take(&mut self.stored_review_comments);
// works for any T: Default — Vec, String, HashMap, BTreeMap
```

For in-place transforms, double-buffer with a pre-allocated spare and swap heap pointers in O(1):

```rust
// write transformed data into `spare_rep` while reading `rep_levels`, then:
std::mem::swap(&mut self.rep_levels, &mut self.spare_rep);
```

| Idiom | Use when |
|-------|----------|
| `mem::take(&mut f)` | move a `T: Default` field out, leave default, avoid clone-then-clear |
| `mem::swap(&mut a, &mut b)` | in-place transform with a reusable spare buffer, zero alloc |

*Source: Zed `crates/editor/src/git.rs` (take); LanceDB `lance-encoding/src/repdef.rs` (swap).*

---

## Anti-Patterns

| Anti-Pattern | Why Bad | Better |
|--------------|---------|--------|
| RefCell everywhere | Runtime panics | Clear ownership design |
| Mutex for single-thread | Unnecessary overhead | RefCell |
| Ignore RefCell panic | Hard to debug | Handle or restructure |
| Lock inside hot loop | Performance killer | Batch operations |

---

## Related Skills

| When | See |
|------|-----|
| Smart pointer choice | m02-resource |
| Thread safety | m07-concurrency |
| Data structure design | m09-domain |
| Anti-patterns | m15-anti-pattern |
