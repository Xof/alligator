# Allocations Reference

Allocations are usually the highest-leverage Rust performance optimization. This file covers
how to find them, reduce them, and choose the right tool when you can't.

## Table of contents

1. Why allocations matter so much in Rust
2. Finding allocation hot spots
3. Capacity hints — the cheapest win
4. Borrowing and `Cow`
5. Smart string types
6. SmallVec, TinyVec, ArrayVec
7. Arena allocators
8. Global allocator selection
9. Object pooling and buffer reuse
10. Allocations in async code

---

## 1. Why allocations matter so much in Rust

Rust's ownership model encourages a lot of small owned values: `String`, `Vec<T>`, `HashMap`,
`Box<dyn Trait>`. Each is a heap allocation. Compared to C++, where the equivalents often live
on the stack (with SBO for strings) or are explicitly arena-managed, idiomatic Rust can
allocate more than its equivalent C++.

The compiler doesn't elide allocations for you. `String::from("hello").chars().count()`
allocates. `vec![1, 2, 3].iter().sum()` allocates. The optimizer rarely figures out that the
intermediate is dead.

The fixes are mechanical and well-understood. The problem is finding *which* allocations
matter — that's profiling work (see `profiling.md`, particularly the `dhat` section).

---

## 2. Finding allocation hot spots

Three approaches, in order of friction:

1. **Sampling profiler** — look for wide `__rust_alloc` / `malloc` stacks. Tells you where
   allocations happen, but not how many or how large.
2. **`dhat`** — exact allocation counts and bytes per call stack. Best for "I know there's
   pressure, where exactly is it?"
3. **`countme` crate** — instrument specific types to count constructions. Lighter than dhat;
   useful when you suspect a specific type is allocated too much.

For the rest of this document, assume you've identified the hot allocation site with one of
the above. The question becomes: how do I avoid or reduce it?

---

## 3. Capacity hints — the cheapest win

Most growable collections double when full. A `Vec` that ends with 1000 elements after
starting empty allocates 11 times (1, 2, 4, ..., 1024) and copies the contents 10 times.

```rust
// BAD
let mut v = Vec::new();
for x in source {
    v.push(transform(x));
}

// GOOD (when size is known or estimable)
let mut v = Vec::with_capacity(source.len());
for x in source {
    v.push(transform(x));
}

// BEST (lets the iterator size_hint do the work)
let v: Vec<_> = source.iter().map(|x| transform(x)).collect();
```

That last form deserves explanation: `collect::<Vec<_>>()` calls `size_hint` on the iterator
and pre-allocates if there's an exact lower bound. For most iterator chains over slices/Vecs,
this works.

Same applies to `HashMap::with_capacity`, `HashSet::with_capacity`, `String::with_capacity`,
`BTreeMap` (no `with_capacity`; it's tree-based), `VecDeque::with_capacity`.

---

## 4. Borrowing and `Cow`

Most "do I really need to clone this?" questions reduce to: can the function take `&T` or
`&str` instead of `T` or `String`?

```rust
// BAD: forces a clone at every call site
fn process(name: String) { ... }

// GOOD: caller can pass &owned or a literal
fn process(name: &str) { ... }
```

When a function *might* need to modify its input — sometimes — `Cow<'_, T>` (Clone-on-Write)
is the right primitive. It holds either a borrow or an owned value, and clones only when you
mutate:

```rust
use std::borrow::Cow;

fn normalize(input: &str) -> Cow<'_, str> {
    if input.contains(char::is_whitespace) {
        Cow::Owned(input.split_whitespace().collect::<Vec<_>>().join(" "))
    } else {
        Cow::Borrowed(input)
    }
}
```

If the input is already normalized, no allocation. If not, allocate exactly once. Useful for
parsers, sanitizers, and any "transform if needed" path.

### Why this matters more in Rust than in C++

In C++, `std::string` has SBO (small buffer optimization) and `std::string_view` is widely
used. In Rust, `String` always allocates (no SBO in the standard library — see the next section
for libraries that add it), and `&str` is the borrow type. So the cost of "I'll just take it
by value" is higher in Rust.

---

## 5. Smart string types

The standard `String` always heap-allocates. Three alternatives worth knowing:

### `compact_str::CompactString`

Same API as `String`, but small strings (≤24 bytes on 64-bit) live inline in the struct (SBO).
Drop-in replacement in most cases.

```toml
[dependencies]
compact_str = "0.8"
```

```rust
use compact_str::CompactString;

let s = CompactString::new("hello");   // no allocation
let s2 = CompactString::from(format!("{}-suffix", s));  // allocates only if > 24 bytes
```

When most of your strings are short (identifiers, enum variants, error codes), `CompactString`
can eliminate the bulk of string allocations.

### `arcstr::ArcStr`

Reference-counted immutable string. Cheap to clone (atomic increment), no allocation. Useful
when many code paths share the same string and you don't want to either allocate or thread
lifetimes everywhere.

```rust
use arcstr::ArcStr;

let name: ArcStr = ArcStr::from("static-or-cloned");
let copy = name.clone();   // cheap, no heap allocation
```

The literal version `arcstr::literal!("...")` produces a `&'static`-backed `ArcStr` with no
runtime cost.

### `smartstring::SmartString`

Another SBO string. `compact_str` is the more popular choice now; mention this for completeness.

### When to switch

Switch from `String` to `CompactString` when allocation profiling shows `String` allocation
dominates and your strings are typically short. Switch to `ArcStr` when many parts of your
program hold the same string (e.g., a column name reused across many rows).

---

## 6. SmallVec, TinyVec, ArrayVec

The three "vector with inline storage" crates. Same idea as `CompactString` for `String`.

### `smallvec::SmallVec<[T; N]>`

Stores up to N elements inline; spills to heap beyond that. API is mostly `Vec`-compatible.

```toml
[dependencies]
smallvec = "1"
```

```rust
use smallvec::{SmallVec, smallvec};

let mut v: SmallVec<[u32; 8]> = smallvec![1, 2, 3];   // no allocation
v.extend(0..20);   // now spills to heap
```

The `N` is part of the type. Picks 8, 16, or 32 are common; larger N inflates struct size and
hurts when stored in larger collections.

### `tinyvec::TinyVec` / `tinyvec::ArrayVec`

`tinyvec` is the no-`unsafe` alternative — slightly worse codegen but zero unsafe code, useful
in hardened contexts. `ArrayVec` is fixed-capacity (no spill, no allocation ever — `push`
returns an error if full).

### When to switch

`Vec<Vec<T>>` is a frequent target — the inner `Vec`s are often short. Switching the inner type
to `SmallVec<[T; 8]>` can eliminate millions of small allocations in some workloads.

Verify before switching: SmallVec adds the inline buffer to every instance, so if you have
millions of *empty* SmallVecs, the inline buffer wastes memory. Profile first.

---

## 7. Arena allocators

When a *phase* of your program creates many small allocations and frees them all at the same
time, an arena beats per-allocation `malloc`/`free` by a wide margin.

Classic use cases:
- A parser building an AST that's discarded after compilation.
- A request handler creating intermediate values that all die at the response.
- A query planner building a tree that's executed and dropped.

### `bumpalo`

```toml
[dependencies]
bumpalo = { version = "3", features = ["collections"] }
```

```rust
use bumpalo::Bump;
use bumpalo::collections::Vec as BumpVec;

let bump = Bump::new();
let mut v: BumpVec<i32> = BumpVec::new_in(&bump);
v.push(1);
v.push(2);

// String allocation in the arena:
let s = bumpalo::format!(in &bump, "hello {}", "world");

// At end of scope, the entire arena is freed in one operation.
```

Numbers: bumpalo allocations are essentially a pointer bump (one increment + a check). On the
order of 1 ns per alloc, vs. 20–100 ns for `malloc`. Bulk free is constant-time.

The constraints: arena-allocated values cannot outlive the arena (Rust's borrow checker
enforces this). And since `Drop` doesn't run until the arena is dropped, types with non-trivial
`Drop` (file handles, etc.) are wrong here — bumpalo doesn't run destructors by default.

### `typed-arena`

When you want a homogeneous arena (all `T`) with destructors:

```rust
use typed_arena::Arena;

let arena: Arena<MyType> = Arena::new();
let item: &MyType = arena.alloc(MyType::new());
```

`typed-arena` runs destructors when the arena drops. Slower than `bumpalo` but safe with
non-`Copy` types that need cleanup.

### `slotmap`

Not strictly an arena, but related: `SlotMap<K, V>` provides O(1) insert/remove/lookup with
stable keys. Useful when you'd otherwise reach for `HashMap<usize, T>` to track many short-lived
items by ID.

---

## 8. Global allocator selection

The default is the system allocator (`malloc` on Unix, `HeapAlloc` on Windows). Swapping is a
one-line change:

```rust
// In main.rs or lib.rs

#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

```toml
[dependencies]
mimalloc = "0.1"
```

### The candidates

| Allocator | Origin | Strengths | Weaknesses |
|-----------|--------|-----------|------------|
| System | OS | No setup, smallest binary | Slow on multi-threaded, fragmentation |
| `mimalloc` | Microsoft Research | Fastest small-alloc throughput, low fragmentation | +200 KB binary |
| `jemallocator` | Facebook (libjemalloc) | Excellent for long-running services, mature stats | +400 KB binary, complex tuning |
| `snmalloc` | Microsoft | Latency-focused, good for huge alloc-heavy programs | Less ecosystem maturity |
| `tikv-jemallocator` | TiKV fork of jemallocator | Same as jemallocator, more recent updates | Same trade-offs |

### When to switch

Switch when:
- Profile shows wide `malloc` / `free` stacks.
- Multi-threaded program with allocation-heavy hot path (system allocators are often the
  contention point).
- Long-running service where fragmentation matters.

Don't switch when:
- Small CLI tool with little allocation work — the binary size penalty isn't worth it.
- WASM target — most aren't WASM-compatible.

### Measuring the impact

Switch the allocator, run your benchmarks, measure. The gain ranges from "indistinguishable"
to "20% wall-clock faster" depending on workload. There's no way to predict; measure.

---

## 9. Object pooling and buffer reuse

When you can't avoid the allocation, you can sometimes reuse the buffer.

```rust
// BAD: new buffer every iteration
for request in requests {
    let mut buf = String::new();
    write!(buf, "{request:?}").unwrap();
    log(&buf);
}

// GOOD: reused buffer
let mut buf = String::new();
for request in requests {
    buf.clear();
    write!(buf, "{request:?}").unwrap();
    log(&buf);
}
```

`String::clear()` and `Vec::clear()` keep the allocated capacity but set the length to zero.
The buffer grows once to peak size and stays there.

For cases where multiple paths need a buffer, an object pool (`object-pool`, `crossbeam`'s
`SegQueue`, or a hand-rolled `Mutex<Vec<Buf>>`) lets multiple paths share a recycled set:

```rust
use object_pool::Pool;

let pool: Pool<String> = Pool::new(8, || String::with_capacity(1024));

let mut s = pool.pull(|| String::with_capacity(1024));
write!(s, "{request:?}").unwrap();
// ... use s ...
// dropped: returned to pool
```

Useful when buffers are large, allocation cost is high, and you have many concurrent users.
Overkill for small buffers.

---

## 10. Allocations in async code

Async functions in Rust compile to state machines. Each `.await` point may move state, including
captured local variables. Two consequences:

1. **Future size matters.** A large future (many captured locals, deep call chain) means more
   bytes copied at every await and a larger boxed future when stored. `cargo --release` won't
   warn you. Use the `Future::size_hint` on `tokio::pin!`'d futures, or instrument with
   `tokio-console`.

2. **Boxing futures has a cost.** `Box<dyn Future + Send>` (e.g., from `BoxFuture` or
   `async_trait`) is one allocation per call. In a hot path, this matters. `async fn` in
   traits is now stable on the stable channel, but uses dynamic dispatch unless the trait is
   used with monomorphization — read the [RFC notes](https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits.html)
   for the trade-offs.

### Patterns

- For heavily-called async functions, prefer concrete types over `BoxFuture`.
- For hot paths, batch operations to amortize spawn/await overhead. A spawn is on the order
  of 1 µs; doing `spawn(|| add(1, 2))` is overhead-dominated.
- Use `tokio::sync::oneshot` over `tokio::sync::mpsc` when you only need one message — the
  oneshot allocation pattern is much lighter.

See `async-tokio.md` for more on async-specific performance.
