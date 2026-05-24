# Antipatterns Catalog

A code-review checklist for Rust performance. Each entry: the smell, why it's bad, the fix,
and how to confirm.

This is meant to be readable end-to-end as a learning tool, and skimmable as a checklist when
reviewing code.

## Categories

1. Allocation antipatterns
2. Iterator antipatterns
3. String antipatterns
4. Concurrency antipatterns
5. Trait/dispatch antipatterns
6. Async antipatterns
7. Build configuration antipatterns
8. Hash table antipatterns
9. Error handling antipatterns
10. Hot-path coding antipatterns

---

## 1. Allocation antipatterns

### 1.1 `Vec::new()` then push without capacity

```rust
// BAD
let mut out = Vec::new();
for x in items {
    out.push(transform(x));
}

// GOOD
let mut out = Vec::with_capacity(items.len());
for x in items {
    out.push(transform(x));
}

// BEST (often)
let out: Vec<_> = items.iter().map(|x| transform(x)).collect();
```

Why: Vec doubles when full. Without a hint, n pushes mean ~log2(n) reallocations and copies.

How to confirm: `dhat` will show the same allocation site with multiple alloc events.

### 1.2 Reflexive `.clone()` to dodge the borrow checker

```rust
// BAD
fn process(name: String) { ... }
process(my_name.clone());

// GOOD
fn process(name: &str) { ... }
process(&my_name);
```

Why: every clone is an allocation. The borrow checker is usually right that you don't need to
own.

How to confirm: search the diff for `.clone()` calls. Each one is a question to answer.

### 1.3 Cloning to create a "snapshot" of a small struct

```rust
// BAD: clones a Vec<String>
let snapshot = state.users.clone();
process(snapshot);
```

Often you can pass `&state.users` and let the function borrow. If the function genuinely needs
to outlive the borrow (e.g., spawned task), consider:
- `Arc<Vec<String>>` — share the existing allocation.
- A precomputed digest if only a fingerprint is needed.

### 1.4 Allocating in a hot loop for logging/debugging

```rust
// BAD
for x in items {
    log::trace!("processing {}", x.detailed_repr());
    process(x);
}
```

Even when logging is filtered out, `format!` may execute its arguments. `log` and `tracing`
both have lazy evaluation if you use the right form:

```rust
// GOOD: only evaluates if level enabled
log::trace!(detailed = ?x, "processing");
```

How to confirm: profile with logs at production level, look for `core::fmt` time.

### 1.5 `Box<T>` for small `T` to avoid generics

```rust
// BAD
struct Wrapper {
    inner: Box<u32>,
}
```

`Box<u32>` is a heap allocation for 4 bytes. Almost never what you want.

The legitimate uses for `Box<T>` of small `T`:
- Polymorphism via `Box<dyn Trait>`.
- Pinning for `Pin<Box<T>>`.
- Recursive types where the size needs to be bounded.

Outside those, prefer `T` directly.

---

## 2. Iterator antipatterns

### 2.1 `.collect::<Vec<_>>()` then iterate again

```rust
// BAD
let items: Vec<_> = source.iter().filter(|x| x.is_active()).collect();
for item in items { use_it(item); }

// GOOD
for item in source.iter().filter(|x| x.is_active()) {
    use_it(item);
}
```

The intermediate Vec is wasted allocation. Iterators are lazy; let them stream.

When this *is* fine: when you need to iterate the result more than once, or need random access
on the result.

### 2.2 `.iter().count()` instead of `.len()`

```rust
// BAD
let n = my_vec.iter().count();   // O(n)

// GOOD
let n = my_vec.len();            // O(1)
```

For any type with O(1) `len()` (Vec, slices, arrays, BTreeMap), use it. `count()` walks the
entire iterator.

### 2.3 `.iter().nth(0)` instead of `.first()`

```rust
// BAD
let head = items.iter().nth(0);

// GOOD
let head = items.first();
```

Similar: `.iter().last()` vs `.last()` (slices have `.last()`).

### 2.4 `.unwrap()` in iterator chains preventing inlining

This isn't an antipattern per se, but worth knowing: long iterator chains with non-trivial
closures can defeat inlining if any closure is too complex. Symptom: profile shows time in
iterator machinery rather than the work. Fix: extract the closure to a named function and
mark `#[inline]`.

### 2.5 `for_each` vs `for` loop

These are equivalent in performance most of the time, but `for_each` consumes its iterator and
sometimes optimizes slightly better with `chain`/`flat_map`. Use whichever reads better; rarely
worth profiling.

---

## 3. String antipatterns

### 3.1 `format!` for simple concatenation

```rust
// BAD
let url = format!("{}/{}", base, path);   // allocation + parsing format string

// GOOD (when the format is trivial)
let mut url = String::with_capacity(base.len() + 1 + path.len());
url.push_str(base);
url.push('/');
url.push_str(path);
```

`format!` is convenient but expensive. For hot paths or simple cases, manual concat with
known capacity wins.

### 3.2 `to_string()` on `&str`

```rust
// BAD if you don't need ownership
let s = "hello".to_string();

// GOOD if you do need ownership and prefer brevity
let s = String::from("hello");

// BEST if you don't need ownership
let s = "hello";   // &'static str
```

`to_string()` and `String::from()` both allocate. Avoid them when `&str` works.

### 3.3 String concatenation in a loop with `+`

```rust
// BAD: allocates a new String each iteration
let mut s = String::new();
for word in words {
    s = s + word;
}

// GOOD: in-place mutation
let mut s = String::new();
for word in words {
    s.push_str(word);
}

// BEST when you know the total size
let total: usize = words.iter().map(|w| w.len()).sum();
let mut s = String::with_capacity(total);
for word in words {
    s.push_str(word);
}
```

The `+` operator on String moves its left side and creates a new String. Each iteration
allocates.

### 3.4 Using `String` as a key in a `HashMap` of constants

```rust
// BAD
let map: HashMap<String, Handler> = HashMap::new();
map.insert("get".to_string(), get_handler);
map.insert("post".to_string(), post_handler);

// Look up:
map.get(&"get".to_string());   // allocates for the lookup!
```

```rust
// GOOD: use &'static str
let map: HashMap<&'static str, Handler> = HashMap::new();
map.insert("get", get_handler);
map.get("get");

// BETTER for compile-time-known sets: phf crate
use phf::phf_map;
static HANDLERS: phf::Map<&'static str, Handler> = phf_map! {
    "get" => get_handler,
    "post" => post_handler,
};
```

`phf` (perfect hash function) generates a static map at compile time with O(1) lookup, no
allocation, no runtime hashing.

---

## 4. Concurrency antipatterns

### 4.1 `Arc<Mutex<T>>` for read-mostly data

```rust
// BAD: every read acquires the lock
struct Server {
    config: Arc<Mutex<Config>>,
}

// GOOD: read-mostly with arc-swap
use arc_swap::ArcSwap;

struct Server {
    config: Arc<ArcSwap<Config>>,
}
// Reads are lock-free atomic loads; writes swap atomically.
```

Or `RwLock` if writes are rare but reads must be consistent with each other.

### 4.2 Holding a `MutexGuard` longer than necessary

```rust
// BAD
{
    let guard = state.lock().unwrap();
    let result = expensive_computation(&guard.data);
    log::info!("result: {result}");
    save_to_db(&guard.data, result).await;   // lock held during all of this
}

// GOOD
let snapshot = {
    let guard = state.lock().unwrap();
    guard.data.clone()    // or extract just what you need
};
let result = expensive_computation(&snapshot);
log::info!("result: {result}");
save_to_db(&snapshot, result).await;
```

The lock should be held for the *minimum* duration. Anything that doesn't need the lock should
move outside the guard's scope.

### 4.3 False sharing in atomic counters

```rust
// BAD: counters share a cache line, every increment invalidates the others
struct Stats {
    counter_a: AtomicU64,
    counter_b: AtomicU64,
    counter_c: AtomicU64,
}

// GOOD: pad to cache-line size
use crossbeam_utils::CachePadded;

struct Stats {
    counter_a: CachePadded<AtomicU64>,
    counter_b: CachePadded<AtomicU64>,
    counter_c: CachePadded<AtomicU64>,
}
```

When multiple threads write to atomics on the same cache line, every write invalidates the
line in every other core's cache. Padding to 64 bytes (or 128 on some architectures) prevents
this.

How to confirm: `perf stat -e cache-misses` will spike on the bad version under contention.

### 4.4 `parking_lot` vs `std::sync`

```rust
// BAD (in low-contention or hot-path code)
use std::sync::Mutex;
let m = Mutex::new(0);
*m.lock().unwrap() += 1;

// GOOD: faster, no poisoning, no unwrap
use parking_lot::Mutex;
let m = Mutex::new(0);
*m.lock() += 1;
```

`parking_lot::Mutex` is faster than `std::sync::Mutex` and skips the poisoning machinery
(which most code unwraps away anyway). Same for `RwLock`. The downside: another dependency,
and you don't get the standard ecosystem's `MutexGuard` type if a third-party API expects it.

For new code, `parking_lot` is the better default.

---

## 5. Trait / dispatch antipatterns

### 5.1 `Box<dyn Trait>` in a tight loop

```rust
// BAD
for handler in handlers.iter() {           // handlers: Vec<Box<dyn Handler>>
    handler.handle(input);                  // virtual dispatch every call
}

// GOOD: enum dispatch
enum AnyHandler {
    Get(GetHandler),
    Post(PostHandler),
}

impl AnyHandler {
    fn handle(&self, input: Input) {
        match self {
            AnyHandler::Get(h) => h.handle(input),
            AnyHandler::Post(h) => h.handle(input),
        }
    }
}
```

The `enum_dispatch` crate generates this from a trait + impl declarations.

When `Box<dyn Trait>` is fine: when the type set is open (plugins, user-extensible), or when
the call cost is dwarfed by the work inside.

### 5.2 Excessive monomorphization

```rust
// CAUSES BLOAT
fn process<T: Display>(x: T) {
    let s = format!("{}", x);
    very_large_function(&s);
}
```

Each distinct `T` creates a copy of `process`, including a copy of `very_large_function`'s
inlined body. With many `T`s, your binary balloons and the instruction cache thrashes.

Fix: extract the non-generic part:

```rust
fn process<T: Display>(x: T) {
    let s = format!("{}", x);
    process_inner(&s);   // not generic
}

fn process_inner(s: &str) {
    very_large_function(s);
}
```

Confirm with `cargo llvm-lines --release | head -40`.

### 5.3 `#[inline(always)]` everywhere

```rust
// BAD
#[inline(always)]
fn helper() { ... }
```

`#[inline(always)]` overrides the compiler's heuristics. Sometimes correct (small numeric
helpers, format specifier dispatch). Often wrong (the function inlines into N callers, each
copy is a miss in the icache, total throughput drops).

The default `#[inline]` (a hint, considered for cross-crate inlining) is right ~95% of the
time. Reach for `#[inline(always)]` only when you've measured both versions.

---

## 6. Async antipatterns

### 6.1 `MutexGuard` across `.await`

(Repeated from `async-tokio.md` — included here for the catalog.)

```rust
// BAD
let guard = state.lock().unwrap();
something_async().await;  // serializes all tasks needing this lock
```

Fix: drop the guard before awaiting, or use `tokio::sync::Mutex`.

### 6.2 `async-trait` in hot paths

```rust
// BAD if called millions of times per second
#[async_trait]
trait Handler {
    async fn handle(&self, req: Req) -> Resp;
}
```

`async-trait` boxes the returned future — one allocation per call. For hot traits, prefer
static dispatch with the now-stable `async fn in trait` syntax, or a manual `Future`-returning
method.

### 6.3 `tokio::spawn` for trivial work

```rust
// BAD
for x in inputs {
    tokio::spawn(async move { add(x, 1) });
}
```

Spawn cost (~1 µs) exceeds the work. Just do the work, or batch.

### 6.4 Sync I/O on the async runtime

```rust
// BAD: blocks the worker
async fn read_config() -> Config {
    let bytes = std::fs::read("config.toml").unwrap();
    parse(bytes)
}

// GOOD
async fn read_config() -> Config {
    let bytes = tokio::fs::read("config.toml").await.unwrap();
    parse(bytes)
}
```

Or for a sync API you can't avoid:

```rust
async fn read_config() -> Config {
    tokio::task::spawn_blocking(|| {
        let bytes = std::fs::read("config.toml").unwrap();
        parse(bytes)
    }).await.unwrap()
}
```

---

## 7. Build configuration antipatterns

### 7.1 Profiling a debug build

```bash
# BAD: numbers are meaningless
cargo run            # debug mode
samply record ./target/debug/my_app

# GOOD
cargo build --profile release-with-debug
samply record ./target/release-with-debug/my_app
```

### 7.2 Benchmarking with `cargo run`

```bash
# BAD
cargo run -- benchmark   # debug mode
hyperfine './target/debug/my_app benchmark'

# GOOD
cargo build --release
hyperfine './target/release/my_app benchmark'
```

### 7.3 No `[profile.release]` tuning

Default release profile is fine for a starter project. For anything performance-sensitive, at
minimum set `lto = "thin"` and consider `codegen-units = 1`. See `compiler-tuning.md`.

### 7.4 Heavy dependencies you don't need

```toml
# BAD: pulls in everything
tokio = { version = "1", features = ["full"] }

# GOOD: only what you use
tokio = { version = "1", features = ["macros", "rt-multi-thread", "net", "io-util"] }
```

Affects compile time, binary size, and indirectly runtime (more code to optimize).

---

## 8. Hash table antipatterns

### 8.1 Default hasher when DoS isn't a concern

```rust
// BAD for performance
use std::collections::HashMap;
let m: HashMap<String, Value> = HashMap::new();   // SipHash-1-3
```

```rust
// GOOD when DoS isn't relevant (internal data, not user input as keys)
use ahash::AHashMap;
let m: AHashMap<String, Value> = AHashMap::new();

// Or even faster
use foldhash::HashMap;
```

The standard hasher is DoS-resistant (cryptographic-ish), and pays for it in performance.
`ahash` and `foldhash` are 2–3× faster on typical workloads. Use them when keys are not
attacker-controlled.

### 8.2 Iterating a `HashMap` with implicit ordering assumptions

```rust
// BAD: HashMap ordering is unspecified and varies
for (k, v) in &my_map {
    write_to_file(k, v);   // file content varies between runs!
}

// GOOD if order matters
let mut sorted: Vec<_> = my_map.iter().collect();
sorted.sort_by_key(|(k, _)| *k);
```

Not strictly a perf issue but causes flaky tests and reproducibility problems.

### 8.3 `HashMap` for tiny sets

```rust
// BAD for small N (say, < 16)
let m: HashMap<u32, u32> = ...;
m.get(&key);

// GOOD for small N
let v: Vec<(u32, u32)> = ...;
v.iter().find(|(k, _)| *k == key).map(|(_, v)| v);
```

Linear scan over a small Vec beats HashMap lookup at small N due to constant factors. Crossover
is around 8–32 items depending on the type. The `linear-map` crate provides a HashMap-shaped
API over a Vec.

---

## 9. Error handling antipatterns

### 9.1 `Result<T, Box<dyn Error>>` in hot paths

```rust
// BAD
fn parse(s: &str) -> Result<Value, Box<dyn Error>> { ... }
```

`Box<dyn Error>` boxes the error variant, every error path allocates. For hot paths,
prefer concrete error types or `thiserror`-generated enums.

When boxing is fine: at the *boundary* (e.g., the top-level result of an operation), where
the cost of the box is dwarfed by the work and where unifying error types is convenient.

### 9.2 `String` errors

```rust
// BAD
fn parse(s: &str) -> Result<Value, String> {
    if !s.is_ascii() {
        return Err(format!("not ascii: {s}"));
    }
    ...
}
```

Every error allocates a String. For hot paths, use a static `&'static str` or an enum:

```rust
enum ParseError {
    NotAscii,
    Empty,
    InvalidChar(char),
}
```

### 9.3 `panic!` in performance-sensitive paths

`panic!` in release with `unwind` strategy is fast but generates code (landing pads, unwind
tables). With `panic = "abort"` it's smaller and slightly faster. Either way, prefer `Result`
in code where the caller might recover.

---

## 10. Hot-path coding antipatterns

### 10.1 Bounds checks not elided

```rust
// May have bounds checks
fn sum(slice: &[u32]) -> u32 {
    let mut total = 0;
    for i in 0..slice.len() {
        total += slice[i];   // bounds check on each iteration
    }
    total
}

// Better: iterator removes bounds checks
fn sum(slice: &[u32]) -> u32 {
    slice.iter().sum()
}
```

Most "you don't need a manual loop" rules in Rust are about elision of bounds checks. Verify
with `cargo asm` — the iterator version should have no `panic_bounds_check` calls.

For cases where bounds checks survive and you've proven they're unnecessary:

```rust
// One-time assertion gives the optimizer a hint
assert!(slice.len() >= n);
for i in 0..n {
    process(slice[i]);   // bounds check often elided after the assert
}
```

`get_unchecked` is the unsafe fallback. Use it only after measurement and only when the safe
version genuinely loses to the unsafe one.

### 10.2 SIMD that didn't actually vectorize

You wrote what you thought was SIMD-friendly code, but the compiler didn't vectorize it.
Confirm with `cargo asm`:
- Look for `vmulps`, `vaddps`, `vfmadd...` (x86 AVX) or `fmla`, `fmul` (ARM NEON).
- If you only see scalar `mulss`, `addss`, vectorization didn't fire.

Common reasons:
- Reduction with non-associative operation (e.g., floating-point sum without `--ffast-math` —
  Rust doesn't have that flag, so floating-point reductions often don't auto-vectorize).
- Pointer aliasing (rust references should help, but unsafe pointer code often doesn't).
- Inner function not inlined.
- Mixed types causing scalar fallback.

Fix: explicit SIMD via `std::simd` (nightly) or the `wide` / `pulp` crates on stable.

### 10.3 Format string parsing in inner loops

```rust
// BAD
for x in items {
    let key = format!("item-{}", x.id);
    map.get(&key);
}

// GOOD: reuse the buffer
let mut key = String::with_capacity(32);
for x in items {
    key.clear();
    write!(key, "item-{}", x.id).unwrap();
    map.get(&key);
}
```

Or precompute the keys if `items` is iterated multiple times.

### 10.4 Repeated `.lock()` per element

```rust
// BAD: lock per element
for x in many_items {
    state.lock().unwrap().insert(x);
}

// GOOD: lock once, batch
{
    let mut guard = state.lock().unwrap();
    for x in many_items {
        guard.insert(x);
    }
}
```

Lock acquisition is cheap but not free. Bulk operations should hold the lock once.
