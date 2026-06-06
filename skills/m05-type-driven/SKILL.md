---
name: m05-type-driven
description: "CRITICAL: Use for type-driven design. Triggers: type state, PhantomData, newtype, marker trait, builder pattern, make invalid states unrepresentable, compile-time validation, sealed trait, ZST, 类型状态, 新类型模式, 类型驱动设计"
user-invocable: false
---

# Type-Driven Design

> **Layer 1: Language Mechanics**

## Core Question

**How can the type system prevent invalid states?**

Before reaching for runtime checks:
- Can the compiler catch this error?
- Can invalid states be unrepresentable?
- Can the type encode the invariant?

---

## Error → Design Question

| Pattern | Don't Just Say | Ask Instead |
|---------|----------------|-------------|
| Primitive obsession | "It's just a string" | What does this value represent? |
| Boolean flags | "Add an is_valid flag" | Can states be types? |
| Optional everywhere | "Check for None" | Is absence really possible? |
| Validation at runtime | "Return Err if invalid" | Can we validate at construction? |

---

## Thinking Prompt

Before adding runtime validation:

1. **Can the type encode the constraint?**
   - Numeric range → bounded types or newtypes
   - Valid states → type state pattern
   - Semantic meaning → newtype

2. **When is validation possible?**
   - At construction → validated newtype
   - At state transition → type state
   - Only at runtime → Result with clear error

3. **Who needs to know the invariant?**
   - Compiler → type-level encoding
   - API users → clear type signatures
   - Runtime only → documentation

---

## Trace Up ↑

When type design is unclear:

```
"Need to validate email format"
    ↑ Ask: Is this a domain value object?
    ↑ Check: m09-domain (Email as Value Object)
    ↑ Check: domain-* (validation requirements)
```

| Situation | Trace To | Question |
|-----------|----------|----------|
| What types to create | m09-domain | What's the domain model? |
| State machine design | m09-domain | What are valid transitions? |
| Marker trait usage | m04-zero-cost | Static or dynamic dispatch? |

---

## Trace Down ↓

From design to implementation:

```
"Need type-safe wrapper for primitives"
    ↓ Newtype: struct UserId(u64);

"Need compile-time state validation"
    ↓ Type State: Connection<Connected>

"Need to track phantom type parameters"
    ↓ PhantomData: PhantomData<T>

"Need capability markers"
    ↓ Marker Trait: trait Validated {}

"Need gradual construction"
    ↓ Builder: Builder::new().field(x).build()
```

---

## Quick Reference

| Pattern | Purpose | Example |
|---------|---------|---------|
| Newtype | Type safety | `struct UserId(u64);` |
| Type State | State machine | `Connection<Connected>` |
| PhantomData | Variance/lifetime | `PhantomData<&'a T>` |
| Marker Trait | Capability flag | `trait Validated {}` |
| Builder | Gradual construction | `Builder::new().name("x").build()` |
| Sealed Trait | Prevent external impl | `mod private { pub trait Sealed {} }` |

## Pattern Examples

### Newtype

```rust
struct Email(String);  // Not just any string

impl Email {
    pub fn new(s: &str) -> Result<Self, ValidationError> {
        // Validate once, trust forever
        validate_email(s)?;
        Ok(Self(s.to_string()))
    }
}
```

**Guard the serde boundary too.** A validated newtype that only checks in `new()` is silently bypassed when the value is *deserialized* — `serde` writes straight into the private field. Route deserialization through the same validator with `#[serde(try_from, into)]`:

```rust
#[derive(Clone, Serialize, Deserialize)]
#[serde(try_from = "String", into = "String")]
pub struct Email(String);

impl TryFrom<String> for Email {
    type Error = ValidationError;
    fn try_from(s: String) -> Result<Self, Self::Error> { Email::new(&s) }
}
impl From<Email> for String { fn from(e: Email) -> String { e.0 } }
```

Now there is *no* path — constructor or wire — that produces an unvalidated `Email`. OpenAI Codex applies exactly this to its `AgentPath` newtype so a deserialized path is validated like a constructed one (`codex-rs/protocol/src/agent_path.rs`):

```rust
#[derive(Clone, Debug, PartialEq, Eq, ..., Serialize, Deserialize, JsonSchema, TS)]
#[serde(try_from = "String", into = "String")]
pub struct AgentPath(String);
```

### Type State

```rust
struct Connection<State>(TcpStream, PhantomData<State>);

struct Disconnected;
struct Connected;
struct Authenticated;

impl Connection<Disconnected> {
    fn connect(self) -> Connection<Connected> { ... }
}

impl Connection<Connected> {
    fn authenticate(self) -> Connection<Authenticated> { ... }
}
```

---

## Decision Guide

| Need | Pattern |
|------|---------|
| Type safety for primitives | Newtype |
| Compile-time state validation | Type State |
| Lifetime/variance markers | PhantomData |
| Capability flags | Marker Trait |
| Gradual construction | Builder |
| Closed set of impls | Sealed Trait |
| Zero-sized type marker | ZST struct |

---

## Anti-Patterns

| Anti-Pattern | Why Bad | Better |
|--------------|---------|--------|
| Boolean flags for states | Runtime errors | Type state |
| String for semantic types | No type safety | Newtype |
| Option for uninitialized | Unclear invariant | Builder |
| Public fields with invariants | Invariant violation | Private + validated new() |
| `_ => ...` catch-all on a domain enum | New variants compile silently, handled wrong | Exhaustive match, no wildcard |

**Exhaustiveness as a compile-time guard.** Encoding states as an enum is only half the win — the other half is making *adding a state* impossible to ignore. Implement `PartialEq`/`match` over the enum with an explicit arm per variant and **omit the `(_, _) => ...` catch-all**. Every new variant then fails to compile in every consumer until it is handled. DataFusion uses this deliberately on its 50+ variant `ScalarValue` so a new type can never get a wrong-but-quiet equality (`datafusion/common/src/scalar/mod.rs`):

```rust
impl PartialEq for ScalarValue {
    fn eq(&self, other: &Self) -> bool {
        use ScalarValue::*;
        // This purposely doesn't have a catch-all "(_, _)" so that
        // any newly added enum variant will require editing this list
        // or else face a compile error
        match (self, other) {
            (Decimal32(v1, p1, s1), Decimal32(v2, p2, s2)) => v1.eq(v2) && p1.eq(p2) && s1.eq(s2),
            (Decimal32(_, _, _), _) => false,
            // ... explicit arm for every variant ...
        }
    }
}
```

---

## Related Skills

| When | See |
|------|-----|
| Domain modeling | m09-domain |
| Trait design | m04-zero-cost |
| Error handling in constructors | m06-error-handling |
| Anti-patterns | m15-anti-pattern |
