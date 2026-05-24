---
name: rust-performance
description: >
  Expert Rust performance analysis and optimization for systems programmers. Trigger on:
  slow Rust code, hot loops, profiling (samply, flamegraph, perf, Instruments, dhat,
  tokio-console), benchmarking (criterion, divan, hyperfine, iai), allocations and
  allocator choice (mimalloc, jemallocator, bumpalo, SmallVec, CompactString, Cow, arena),
  build tuning (LTO, PGO, target-cpu, codegen-units, panic=abort), dispatch
  (monomorphization, Box dyn, enum_dispatch), hashing (ahash, foldhash, dashmap),
  concurrency (parking_lot, rayon, tokio runtime, spawn_blocking, MutexGuard across await,
  false sharing), low-level work (bounds checks, SIMD, inline, cache misses), binary
  inspection (cargo-bloat, cargo-asm, cargo-llvm-lines). Also trigger on "review this Rust
  for performance", "why is this slow", "reduce allocations", "tune the release profile",
  or "set up benchmarks". Use proactively when reviewing Rust in performance-sensitive
  contexts (database engines, high-throughput services). Always measure first.
---

# Rust Performance Skill

Audience: experienced systems programmers comfortable with C/C++ performance work who want
the Rust-specific equivalents and gotchas. Skip basics. Be direct. Always favor measurement
over intuition.

## Core Principles

- **Measure, do not guess.** Microbenchmark with `criterion` or `divan`. Time end-to-end with
  `hyperfine`. Profile with `samply` (cross-platform), `cargo flamegraph` (Linux/macOS), or
  Xcode Instruments (macOS). Never recommend a change without before/after numbers.
- **Build in release mode for any measurement.** Debug builds are 10–100× slower and have
  different inlining, monomorphization, and bounds-check behavior. Microbenchmarks measured
  in debug mode are worthless.
- **Defeat the optimizer in microbenchmarks.** Use `std::hint::black_box(...)` around inputs
  and outputs, otherwise the compiler will hoist, fold, or eliminate the work entirely.
- **Profile-driven optimization only.** Even experienced engineers are wrong about hot spots
  ~30% of the time. Profile first, then optimize the actual bottleneck.
- **Distinguish throughput from tail latency.** A throughput optimization (batching, larger
  buffers) often makes p99 worse. Decide which metric matters before tuning.
- **Beware the symbol-stripping trap.** Profilers need debug symbols. Use a `release-with-debug`
  profile (see [Compiler Configuration](#compiler-configuration)) for any profiled build.

---

## Workflow Selector

Pick the right mode based on the user's question.

| Situation | Mode | Where to start |
|----------|------|----------------|
| "This is slow, why?" | **Forensic** | [Forensic Workflow](#forensic-workflow) |
| "Review this for performance" | **Preventive** | [Antipattern Checklist](#antipattern-checklist) |
| "Help me design X for performance" | **Architectural** | [Architectural Decisions](#architectural-decisions) |
| "Set up benchmarks/profiling for this project" | **Tooling setup** | `references/benchmarking.md`, `references/profiling.md` |
| "Tune the release profile" | **Build config** | [Compiler Configuration](#compiler-configuration) |

---

## Forensic Workflow

When the user says "this is slow":

1. **Reproduce with a measurement.** Either a `criterion` bench or a `hyperfine` run. If
   neither exists yet, write the bench *first*. Without a number, there is nothing to optimize
   against. See `references/benchmarking.md`.
2. **Confirm release mode.** `cargo bench` and `cargo build --release` are mandatory. If the
   user is timing `cargo run` without `--release`, that is the entire problem; stop here.
3. **Profile.** Pick the right profiler for the platform — see [Profiler Selection](#profiler-selection).
   Generate a flamegraph or sampling profile. Look for the widest stack, not the deepest.
4. **Form a hypothesis from the profile.** Common shapes:
   - Wide `__rust_alloc` / `malloc` stack → allocation pressure. Go to `references/allocations.md`.
   - Wide `memcpy` → unnecessary cloning or large value moves. Search for `.clone()`, `.to_vec()`, `.to_string()`.
   - Wide `format_args` / `core::fmt` → string formatting in hot path. Replace with `write!` to
     a reused buffer or precomputed string.
   - Wide `_$LT$alloc..vec..Vec$LT$T$GT$$u20$as$u20$core..iter` → iterator collecting; can it
     be streamed instead?
   - Wide hash table operations → bad hasher (default `SipHash` is DoS-resistant but slow);
     swap in `ahash` or `foldhash` if DoS is not a threat.
   - Wide tokio runtime / `park_timeout` → either too few worker threads, blocking work on the
     async runtime, or lock contention. See `references/async-tokio.md`.
   - Even spread with no obvious peak → likely cache-bound or branch-prediction-bound; consider
     data layout, SoA vs AoS, or SIMD.
5. **Make one change.** Re-measure. Keep a running log of (change, before, after).
6. **Stop when the bench meets the goal.** Diminishing returns are real; don't optimize past
   the requirement.

### Profiler Selection

| Platform | First choice | Notes |
|----------|--------------|-------|
| macOS (Apple Silicon or Intel) | `samply` | `cargo install samply`, then `samply record ./target/release/foo`. Opens Firefox Profiler in browser. No DTrace privilege issues. |
| macOS, deeper analysis | Xcode Instruments | "Time Profiler" template. Excellent for Allocations, System Trace, Counters templates. |
| Linux | `samply` or `perf` | `samply` for ease; `perf` + `cargo flamegraph` for full power. |
| Heap profile (any OS) | `dhat` crate | Programmatic — wrap your `main` and run normally. Excellent for finding *which* code paths allocate. |
| Heap profile (Linux GUI) | `heaptrack` | Heavier setup, much better visualization than dhat for large traces. |
| Async/tokio diagnostics | `tokio-console` | Requires `console-subscriber`. Shows task lifetimes, busy/idle ratios, polling latencies. |

Full guide: `references/profiling.md`.

---

## Antipattern Checklist

When asked to review Rust code for performance, walk this list. Most real Rust performance
issues are on it.

### Allocation antipatterns

- `format!()`, `to_string()`, `String::from()` in a hot loop — use a reused `String` buffer
  and `write!()`.
- `Vec::new()` followed by repeated `push` when the size is known — use `Vec::with_capacity(n)`.
  Same for `HashMap::with_capacity`, `String::with_capacity`.
- `.clone()` on an owned value when a `&` or `Cow` would suffice. Particularly suspicious:
  cloning inside `.map()` or `.iter()`.
- `String` map keys when `&str` would do — `HashMap<String, V>` should usually be
  `HashMap<&'a str, V>` or use a hash of the string.
- `Vec<Vec<T>>` with many short inner vecs — switch to a flat `Vec<T>` with offsets, or to
  `Vec<SmallVec<[T; N]>>`.
- `Box<dyn Trait>` in a tight loop when an enum or generic would monomorphize. Each call is
  a virtual dispatch *and* defeats inlining.
- Default `HashMap` hasher (`SipHash`) when DoS is not a concern — replace with
  `ahash::AHashMap` or `foldhash::HashMap` for a 2–3× hash speedup.

### Async / concurrency antipatterns

- Holding a `MutexGuard` (or `RefCell` borrow) across `.await`. Serializes the runtime, may
  deadlock, may panic. Use `tokio::sync::Mutex` for cross-await locking, but better: restructure
  to release the lock before awaiting.
- `Arc<Mutex<T>>` for read-mostly data. Use `arc_swap::ArcSwap`, `RwLock`, or `papaya` (lock-free
  hash map) instead.
- `spawn` for trivial async work. The spawn cost (~1µs) can exceed the work itself.
- Sync I/O on the async runtime. Use `spawn_blocking`, or move to a sync architecture if all
  the I/O is sync.
- Default Tokio `blocking` pool size (512). Almost always too high; tune down or use a
  dedicated pool.

### Cross-cutting antipatterns

- No `[profile.release]` tuning at all. At minimum set `lto = "thin"` and consider
  `codegen-units = 1` for production builds. See [Compiler Configuration](#compiler-configuration).
- Calling `.collect::<Vec<_>>()` then iterating again — chain the iterator instead.
- `.iter().count()` instead of `.len()` on a slice/Vec.
- Excessive trait-object indirection across layers — costs both dispatch and inlining.
- Excessive monomorphization (huge generics deeply instantiated) — bloats binary, blows the
  instruction cache, slows compile times. Use `#[inline(never)]` on the cold paths or extract
  a non-generic inner function.
- `derive(Clone)` propagated through structs that never get cloned in the hot path — verify
  with profiler before assuming it's free.

Full catalog with code examples and fixes: `references/antipatterns.md`.

---

## Architectural Decisions

When the user is choosing between approaches *before* writing code, surface these axes
explicitly.

### Allocator selection

The default Rust allocator is the system allocator (`malloc` on Unix, `HeapAlloc` on Windows).
On allocation-heavy workloads, swapping it is one of the cheapest wins available. Set it
once in `main.rs`:

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

| Allocator | When | Trade-off |
|-----------|------|-----------|
| System default | Single-threaded, low-allocation | No setup, no extra binary size |
| `mimalloc` | General purpose, multi-threaded | ~+200KB binary; usually 5–15% wall-clock win |
| `jemallocator` | Long-running services with fragmentation concerns | Larger binary; excellent for servers |
| `snmalloc` | Latency-sensitive | Less battle-tested in Rust ecosystem |
| `bumpalo` arena | Phase-based work (parsing, request handling) | Bulk free at end of phase; not a global |

Full guide: `references/allocations.md`.

### Async runtime

- **Multi-threaded Tokio (default)**: best for general I/O-heavy workloads. Default worker
  count = num CPUs.
- **Single-threaded Tokio (`current_thread`)**: lower overhead, no synchronization between
  workers, but cannot saturate multiple cores. Good for: per-core sharded designs, edge
  workloads, embedded scenarios.
- **`async-std` / `smol`**: rarely justified now that Tokio dominates. Only consider for
  specific runtime characteristics or to avoid Tokio dependency weight.

### Dispatch strategy

For polymorphism in hot paths:

1. **Generics** (monomorphization) — fastest at runtime, slowest to compile, biggest binary.
2. **Enum dispatch** — `match` on a closed set of variants, fully inlinable. Sweet spot when
   the variant set is known.
3. **`Box<dyn Trait>`** — easiest to write, runtime dispatch cost (~1ns per call), defeats
   inlining. Use when the type set is open or unknown at compile time.

The `enum_dispatch` crate generates 2 from 3 ergonomically.

### Data layout

- **Struct of Arrays (SoA)** vs **Array of Structs (AoS)**: SoA wins when iterating one or two
  fields at a time across many instances (cache locality). AoS wins when accessing all fields
  of one instance at a time. The `soa_derive` and `soa-rs` crates make SoA ergonomic.
- **Padding and alignment**: `#[repr(C)]` for predictable layout, `#[repr(packed)]` to remove
  padding (warning: misaligned access on some architectures). `mem::size_of` and
  `mem::align_of` to verify.
- **Cache line awareness**: 64 bytes on most modern CPUs. Use `crossbeam_utils::CachePadded`
  to prevent false sharing on hot atomic counters.

---

## Compiler Configuration

A reasonable default `Cargo.toml` for performance-sensitive builds:

```toml
[profile.release]
opt-level = 3
lto = "fat"            # whole-program optimization across crates
codegen-units = 1      # one CGU = best optimization, slowest compile
panic = "abort"        # smaller binary, no unwind tables, faster
strip = "symbols"      # smaller binary

[profile.release-with-debug]
inherits = "release"
debug = "line-tables-only"
strip = "none"
```

Use `release-with-debug` for any profiled build so the profiler can resolve symbols.

### Trade-offs explained

- **`lto = "fat"`** vs **`lto = "thin"`**: `fat` gives the best optimization (cross-crate
  inlining, dead code elimination across the whole program) but compile time and memory
  balloon. `thin` is a parallel-friendly compromise — usually 80% of the win at 20% of the
  cost. Production: `fat`. Iterating on a benchmark: `thin`.
- **`codegen-units = 1`**: forces single-threaded codegen, enables maximum cross-function
  optimization within a crate. Compile time penalty is significant. Default is 16 for release.
- **`panic = "abort"`**: removes unwind tables (smaller binary, faster), but disables
  `catch_unwind`. Almost always the right choice for binaries; libraries should leave it alone.
- **`target-cpu=native`**: enable via `RUSTFLAGS="-C target-cpu=native"` for binaries that will
  only run on the build machine. Enables AVX2/AVX-512/NEON instructions the default
  baseline doesn't use. Trade-off: binary won't run on older CPUs.

### PGO (Profile-Guided Optimization)

Workflow: build with profiling instrumentation → run representative workload → rebuild with
profile data. Typical real-world gains: 5–15%. Use `cargo-pgo`:

```bash
cargo install cargo-pgo
cargo pgo build
cargo pgo run -- <representative workload>
cargo pgo optimize
```

Full guide: `references/compiler-tuning.md`.

---

## Quick Reference: Tools to Install

For a project with serious performance focus, install these once:

```bash
# Benchmarking & timing
cargo install hyperfine
# (criterion and divan are added as dev-dependencies, not installed)

# Profiling
cargo install samply
cargo install flamegraph         # uses perf on Linux, dtrace on macOS
cargo install cargo-pgo          # PGO

# Inspection
cargo install cargo-bloat        # what's eating the binary
cargo install cargo-asm          # see generated assembly per function
cargo install cargo-show-asm     # newer alternative to cargo-asm
cargo install cargo-llvm-lines   # find monomorphization bloat

# Allocation tracking (added as dev-dependency)
# dhat, dhat-heap
```

For Linux servers also: `perf`, `heaptrack`, `valgrind` (system packages).

---

## Reference Files

- `references/benchmarking.md` — `criterion`, `divan`, `hyperfine`, statistical pitfalls,
  warmup, `black_box`, throughput vs latency benches.
- `references/profiling.md` — Per-platform profiler walkthroughs (`samply`, `cargo flamegraph`,
  Instruments, `perf`, `dhat`, `tokio-console`), interpreting flamegraphs, common stack shapes.
- `references/compiler-tuning.md` — Full release profile reference, LTO modes, PGO and BOLT,
  `target-cpu` and `target-feature`, panic strategies, codegen-units, debug-in-release patterns.
- `references/allocations.md` — Allocator selection, smart string types (`CompactString`,
  `SmartString`, `arcstr`), `SmallVec`/`TinyVec`/`ArrayVec`, arena allocators
  (`bumpalo`, `typed-arena`), `Cow` patterns, capacity hints, allocator-aware collections.
- `references/async-tokio.md` — Runtime tuning, worker count, blocking pool, `tokio-console`,
  span overhead, Mutex-across-await, `Arc<Mutex>` alternatives, channel selection,
  cancellation safety.
- `references/antipatterns.md` — Exhaustive checklist with code examples, before/after, and
  reasoning. Use as a code review aid.
