# Compiler Tuning Reference

Everything that goes in `[profile.*]` and `RUSTFLAGS`, with the trade-offs explained.

## Table of contents

1. The release profile, in detail
2. `lto` modes
3. `codegen-units`
4. `panic` strategies
5. `opt-level` choices (incl. `s`, `z`, `1`)
6. Debug symbols in release
7. `target-cpu` and `target-feature`
8. Profile-Guided Optimization (PGO)
9. BOLT (post-link optimization)
10. Diagnosing what the compiler did
11. Reducing binary size
12. Reducing compile time without giving up performance

---

## 1. The release profile, in detail

A reasonable starting point for production binaries:

```toml
[profile.release]
opt-level = 3
lto = "fat"
codegen-units = 1
panic = "abort"
strip = "symbols"
debug = false
overflow-checks = false   # default in release; explicit for clarity
incremental = false       # default in release; explicit for clarity

[profile.release-with-debug]
inherits = "release"
debug = "line-tables-only"
strip = "none"
```

Each setting matters; let's go through them.

---

## 2. `lto` modes

LTO (Link-Time Optimization) lets the compiler optimize across crate boundaries — most
importantly, inline and dead-code-eliminate functions defined in dependencies.

| Setting | Behavior | Compile time | Runtime gain |
|---------|----------|-------------|--------------|
| `lto = false` (default for release) | Per-crate optimization only | Fastest | Baseline |
| `lto = "thin"` | Parallel cross-crate LTO via LLVM ThinLTO | +30–80% | Often most of the win |
| `lto = "fat"` (or `true`) | Single-threaded whole-program LTO | +100–300% | Maximum |
| `lto = "off"` | Explicitly disabled | Fastest | Baseline |

**When does LTO matter most?** When your program calls many small functions defined in other
crates, especially in tight loops. A common pattern: a Rust binary that uses `serde`, `tokio`,
or any heavy framework — LTO can inline and prune significant overhead.

**When does it not?** A program that does most of its work in a single crate's own hot loops
gains less, since intra-crate optimization already happens.

### A word on the costs of `lto = "fat"`

- Single-threaded link step. Building a large workspace can add minutes, not seconds.
- LLVM memory use during LTO can be massive (multi-GB). 8 GB machines may OOM on big
  workspaces.
- Iteration becomes painful. Use `[profile.release]` LTO for shipping; iterate against
  `[profile.bench]` or a custom `[profile.release-fast]` with `lto = "thin"`.

---

## 3. `codegen-units`

LLVM splits each crate into N "codegen units" (CGUs) for parallel compilation. Optimization
quality drops slightly when split: cross-CGU inlining and analysis is limited.

| Setting | Behavior |
|---------|----------|
| `codegen-units = 1` | Best optimization, single-threaded codegen for the crate |
| `codegen-units = 16` (release default) | Balanced — fast compile, slight perf cost |
| `codegen-units = 256` (debug default) | Maximum parallelism, no optimization |

For shipping builds, set `codegen-units = 1`. Combined with `lto = "fat"`, this gives the
compiler the most freedom.

---

## 4. `panic` strategies

| Setting | Behavior |
|---------|----------|
| `panic = "unwind"` (default) | Stack unwinds on panic; destructors run; `catch_unwind` works |
| `panic = "abort"` | Process exits immediately on panic; no unwind tables; smaller binary |

`abort` saves binary size (no unwinding metadata) and gives a small runtime improvement (no
landing pads in generated code). Trade-off:

- `catch_unwind` becomes a no-op (never catches). Don't use this if any dependency relies on
  catching panics — notable example: some test frameworks, FFI boundaries that need to convert
  panics to error codes.
- The panic message and location still print before abort, so debugging isn't worse.

For **binaries**: usually set `panic = "abort"`. For **libraries**: leave the default; let the
final binary decide.

Note: `panic = "abort"` in `[profile.dev]` disables `cargo test`'s panic-catching, breaking the
test runner. Apply it in `[profile.release]` only.

---

## 5. `opt-level` choices

| Value | Meaning |
|-------|---------|
| `0` | No optimization (debug default). Fastest compile. |
| `1` | Basic optimizations. Often enough to make development sessions usable; pairs well with `[profile.dev.package."*"] opt-level = 1` to keep dependencies fast. |
| `2` | Most optimizations. |
| `3` | All optimizations including aggressive auto-vectorization (release default). |
| `"s"` | Optimize for binary size — moderate optimizations, prefer smaller code. |
| `"z"` | Optimize for binary size aggressively — sacrifices speed. |

For performance work, `3` is the only choice for release. `s` and `z` matter for embedded,
WebAssembly, and CLI tools where startup time and binary size dominate.

A useful trick for development:

```toml
[profile.dev.package."*"]
opt-level = 1
```

This keeps your own crate fast to compile but optimizes dependencies — often eliminates the
"my dev build is unusable because serde_json is in O0" pain.

---

## 6. Debug symbols in release

Debug symbols (`debug = true`) add binary size but enable profiler stack resolution. Three
useful options:

| Setting | What you get |
|---------|--------------|
| `debug = false` | No debug info. Profilers show `[unknown]` or addresses. |
| `debug = "line-tables-only"` | File and line numbers, no full DWARF. Symbol names visible in profilers. |
| `debug = true` (or `"full"`) | Full DWARF. Largest binary; needed for source-level debugging. |

For production binaries you intend to profile in the field: `debug = "line-tables-only"` plus
keep symbols (`strip = "none"` or `strip = "debuginfo"`). The runtime cost is zero; the binary
is somewhat larger.

The `release-with-debug` pattern lets you ship a small binary while keeping a debuggable
build for profiling sessions.

---

## 7. `target-cpu` and `target-feature`

By default, rustc generates code that runs on a baseline x86_64 (SSE2-era) or aarch64 (ARMv8.0)
CPU. This is portable but leaves modern instructions unused.

### `target-cpu=native`

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --release
```

Generates code tuned for the build machine's exact CPU. Enables AVX2, AVX-512, NEON variants,
etc., as available. Trade-off: **the binary will not run on older or different CPUs**.

Right for: self-hosted services, internal tooling, scientific computing where the binary runs
on known hardware.

Wrong for: anything you distribute.

### Specific microarchitectures

If you know your target CPU but it's not the build machine:

```bash
RUSTFLAGS="-C target-cpu=znver3" cargo build --release      # AMD Zen 3
RUSTFLAGS="-C target-cpu=skylake-avx512" cargo build --release
RUSTFLAGS="-C target-cpu=apple-m1" cargo build --release    # not always recognized; check rustc --print target-cpus
```

### `target-feature`

For finer control — enable specific instructions without committing to a microarchitecture:

```bash
RUSTFLAGS="-C target-feature=+avx2,+fma" cargo build --release
```

Useful when you want AVX2 across an x86_64 distribution but want to be clear about minimum
requirements. Pair with runtime detection (`is_x86_feature_detected!`) for fallback paths.

### Distributed binaries with multiple variants

For shipping a single binary with feature dispatch:
- The `multiversion` crate generates multiple variants of a function and dispatches at runtime.
- For full multi-variant binaries, build several artifacts and select via a launcher.

---

## 8. Profile-Guided Optimization (PGO)

PGO collects branch-prediction and call-frequency data from a representative run, then
recompiles with that data. Typical gain: 5–15% on real workloads. Larger on branch-heavy code.

### Workflow with `cargo-pgo`

```bash
cargo install cargo-pgo

# Step 1: Build with instrumentation
cargo pgo build

# Step 2: Run the instrumented binary against representative input
cargo pgo run -- <args matching your real workload>
# Or for benchmarks:
cargo pgo bench

# Step 3: Rebuild using the collected profile
cargo pgo optimize
```

The "representative input" matters — PGO trained on a workload very different from production
can leave you no better off, occasionally slightly worse.

### Manual PGO workflow (no cargo-pgo)

```bash
# Instrumented build
RUSTFLAGS="-Cprofile-generate=/tmp/pgo-data" cargo build --release

# Run representative workload
./target/release/my_app <args>

# Merge profile data
llvm-profdata merge -o /tmp/pgo-data/merged.profdata /tmp/pgo-data

# Rebuild using profile
RUSTFLAGS="-Cprofile-use=/tmp/pgo-data/merged.profdata" cargo build --release
```

### When PGO is worth it

- Production binaries that run for a long time on the same workload.
- Branch-heavy code (parsers, interpreters, query planners).
- Anything dispatching through `match` or virtual calls in hot loops.

### When it isn't

- Microbenchmarks (the bench *is* the training, defeats the purpose).
- Code dominated by external work (I/O, syscalls, GPU).
- Iteration loops — PGO adds a multi-step build.

---

## 9. BOLT (post-link optimization)

BOLT is LLVM's post-link optimizer. It rearranges code blocks based on a *separate* profile
collected after PGO, producing further wins of a few percent. Originally from Facebook, now
in upstream LLVM.

Used in production at Meta and a few other places. Setup is non-trivial: you need
`llvm-bolt`, you build with PGO, you run the binary under perf, you feed the perf output to
BOLT, it rewrites the binary.

Worth knowing exists; rarely worth using outside very large, latency-critical services. The
`cargo-pgo` tool can drive a BOLT step (`cargo pgo bolt`).

---

## 10. Diagnosing what the compiler did

When you suspect an optimization didn't fire:

```bash
# What's in my binary, by crate?
cargo install cargo-bloat
cargo bloat --release
cargo bloat --release --crates

# How much LLVM IR did each generic monomorphization generate?
cargo install cargo-llvm-lines
cargo llvm-lines --release | head -40

# What's the assembly for a specific function?
cargo install cargo-show-asm
cargo asm --release my_crate::my_function
```

`cargo-llvm-lines` is particularly useful for spotting monomorphization bloat — a generic
function that's instantiated 50 times can dwarf the rest of the binary.

---

## 11. Reducing binary size

If binary size matters (CLI tools, WASM, embedded):

```toml
[profile.release]
opt-level = "z"           # or "s"
lto = "fat"
codegen-units = 1
panic = "abort"
strip = "symbols"
```

Plus:

```bash
# Identify the bloat
cargo bloat --release --crates
cargo llvm-lines --release | head -40

# Common offenders
# - serde derives (each Serialize/Deserialize impl is sizable)
# - tokio's "full" feature
# - regex (use regex-lite for smaller binaries)
# - chrono (use jiff or time for smaller binaries)
```

For WASM specifically, `wasm-opt` (binaryen) post-processes for additional wins:

```bash
wasm-opt -Oz -o output.wasm input.wasm
```

---

## 12. Reducing compile time without giving up performance

A common trap: production performance settings make iteration unbearable. Solutions:

- **Tiered profiles**:
  ```toml
  [profile.dev]
  opt-level = 0
  debug = true

  [profile.dev.package."*"]
  opt-level = 1            # fast deps, debug-mode own code

  [profile.release-fast]
  inherits = "release"
  lto = "thin"             # not "fat"
  codegen-units = 16       # not 1

  [profile.release]
  lto = "fat"
  codegen-units = 1
  ```

  Bench against `release-fast` during iteration; switch to `release` only for the final
  before-and-after.

- **`sccache`** caches compilation across builds (and across machines if you use a shared
  backend). For CI: huge wins. For local: depends on your iteration pattern.

- **mold** linker on Linux (`cargo install --locked --git ... mold`) cuts link time
  dramatically for big binaries:
  ```toml
  # .cargo/config.toml
  [target.x86_64-unknown-linux-gnu]
  linker = "clang"
  rustflags = ["-C", "link-arg=-fuse-ld=mold"]
  ```

- **`cranelift` codegen backend** for debug builds — much faster to compile than LLVM, no
  optimization. Available on nightly. Useful when most of your iteration is in debug mode and
  rebuild speed matters more than runtime.
