---
name: m02-resource
description: "CRITICAL: Use for smart pointers and resource management. Triggers: Box, Rc, Arc, Weak, RefCell, Cell, smart pointer, heap allocation, reference counting, RAII, Drop, should I use Box or Rc, when to use Arc vs Rc, 智能指针, 引用计数, 堆分配"
user-invocable: false
---

# Resource Management

> **Layer 1: Language Mechanics**

## Core Question

**What ownership pattern does this resource need?**

Before choosing a smart pointer, understand:
- Is ownership single or shared?
- Is access single-threaded or multi-threaded?
- Are there potential cycles?

---

## Error → Design Question

| Error | Don't Just Say | Ask Instead |
|-------|----------------|-------------|
| "Need heap allocation" | "Use Box" | Why can't this be on stack? |
| Rc memory leak | "Use Weak" | Is the cycle necessary in design? |
| RefCell panic | "Use try_borrow" | Is runtime check the right approach? |
| Arc overhead complaint | "Accept it" | Is multi-thread access actually needed? |

---

## Thinking Prompt

Before choosing a smart pointer:

1. **What's the ownership model?**
   - Single owner → Box or owned value
   - Shared ownership → Rc/Arc
   - Weak reference → Weak

2. **What's the thread context?**
   - Single-thread → Rc, Cell, RefCell
   - Multi-thread → Arc, Mutex, RwLock

3. **Are there cycles?**
   - Yes → One direction must be Weak
   - No → Regular Rc/Arc is fine

---

## Trace Up ↑

When pointer choice is unclear, trace to design:

```
"Should I use Arc or Rc?"
    ↑ Ask: Is this data shared across threads?
    ↑ Check: m07-concurrency (thread model)
    ↑ Check: domain-* (performance constraints)
```

| Situation | Trace To | Question |
|-----------|----------|----------|
| Rc vs Arc confusion | m07-concurrency | What's the concurrency model? |
| RefCell panics | m03-mutability | Is interior mutability right here? |
| Memory leaks | m12-lifecycle | Where should cleanup happen? |

---

## Trace Down ↓

From design to implementation:

```
"Need single-owner heap data"
    ↓ Use: Box<T>

"Need shared immutable data (single-thread)"
    ↓ Use: Rc<T>

"Need shared immutable data (multi-thread)"
    ↓ Use: Arc<T>

"Need to break reference cycle"
    ↓ Use: Weak<T>

"Need shared mutable data"
    ↓ Single-thread: Rc<RefCell<T>>
    ↓ Multi-thread: Arc<Mutex<T>> or Arc<RwLock<T>>
```

### Constructing a cycle (the chicken-and-egg problem)

"Use Weak for one direction" raises a build-order question: the parent doesn't exist yet when the child needs a `Weak` pointing back to it. `Rc::new_cyclic` (stable since 1.60) hands you a `Weak<T>` to the not-yet-initialized `Rc<T>` inside the constructor closure — no `Option<Weak<T>>` field you mutate later, no `unsafe`:

```rust
// Zed — crates/gpui/src/app.rs:702-767
let app = Rc::new_cyclic(|this| AppCell {
    app: RefCell::new(App {
        this: this.clone(),       // <-- The App holds Weak<AppCell> back to itself
        platform: platform.clone(),
        ...
    }),
});
```

`Arc::new_cyclic` is the multi-thread twin. Reach for this the moment a node has to store a back-reference to its own container. (The `this` weak can't be `upgrade()`d inside the closure — the `Rc` isn't finished; only clone/save it.)

---

## Quick Reference

| Type | Ownership | Thread-Safe | Use When |
|------|-----------|-------------|----------|
| `Box<T>` | Single | Yes | Heap allocation, recursive types |
| `Rc<T>` | Shared | No | Single-thread shared ownership |
| `Arc<T>` | Shared | Yes | Multi-thread shared ownership |
| `Weak<T>` | Weak ref | Same as Rc/Arc | Break reference cycles |
| `Cell<T>` | Single | No | Interior mutability (Copy types) |
| `RefCell<T>` | Single | No | Interior mutability (runtime check) |

> **Using a `Weak`:** a `Weak` doesn't keep the target alive, so access is fallible. Store the `Weak`, then call `.upgrade()` at each use site — it returns `Option<Rc<T>>` (or `Option<Arc<T>>`), `None` once the last strong ref is gone. Codex breaks an `Arc` ownership cycle (`ThreadManagerState → CodexThread → Session → … → ThreadManagerState`) by storing `manager: Weak<ThreadManagerState>` and asserting liveness via `upgrade()` inside methods:
>
> ```rust
> // OpenAI Codex — codex-rs/core/src/agent/control.rs
> struct AgentControl {
>     manager: Weak<ThreadManagerState>, // Weak to avoid the cycle above
>     state: Arc<AgentRegistry>,
> }
> // ...at each use site, upgrade-or-bail:
> let state = self.upgrade()?;          // Option<Arc<ThreadManagerState>>; None ⇒ manager dropped
> ```

## Decision Flowchart

```
Need heap allocation?
├─ Yes → Single owner?
│        ├─ Yes → Box<T>
│        └─ No → Multi-thread?
│                ├─ Yes → Arc<T>
│                └─ No → Rc<T>
└─ No → Stack allocation (default)

Have reference cycles?
├─ Yes → Use Weak for one direction
└─ No → Regular Rc/Arc

Need interior mutability?
├─ Yes → Thread-safe needed?
│        ├─ Yes → Mutex<T> or RwLock<T>
│        └─ No → T: Copy? → Cell<T> : RefCell<T>
└─ No → Use &mut T
```

---

## Common Errors

| Problem | Cause | Fix |
|---------|-------|-----|
| Rc cycle leak | Mutual strong refs | Use Weak for one direction |
| RefCell panic | Borrow conflict at runtime | Use try_borrow or restructure |
| Arc overhead | Atomic ops in hot path | Consider Rc if single-threaded |
| Box unnecessary | Data fits on stack | Remove Box |

---

## Anti-Patterns

| Anti-Pattern | Why Bad | Better |
|--------------|---------|--------|
| Arc everywhere | Unnecessary atomic overhead | Use Rc for single-thread |
| RefCell everywhere | Runtime panics | Design clear ownership |
| Box for small types | Unnecessary allocation | Stack allocation |
| Ignore Weak for cycles | Memory leaks | Design parent-child with Weak |

---

## Related Skills

| When | See |
|------|-----|
| Ownership errors | m01-ownership |
| Interior mutability details | m03-mutability |
| Multi-thread context | m07-concurrency |
| Resource lifecycle | m12-lifecycle |
