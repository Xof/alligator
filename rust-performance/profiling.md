# Profiling Reference

Per-platform walkthroughs for the profilers worth knowing, plus how to read the output.

## Table of contents

1. The principle: sampling vs instrumentation
2. samply (cross-platform, recommended starting point)
3. cargo flamegraph (Linux/macOS)
4. Xcode Instruments (macOS)
5. perf (Linux)
6. dhat (heap profiling, any OS)
7. heaptrack (Linux heap profiling with GUI)
8. tokio-console (async runtime diagnostics)
9. cargo-asm / cargo-show-asm (function-level assembly)
10. Reading flamegraphs
11. Common stack shapes and what they mean

---

## 1. Sampling vs instrumentation

- **Sampling profilers** (samply, perf, Instruments Time Profiler) interrupt the program
  periodically and record the stack. Low overhead (~1%), statistical, the right default.
- **Instrumented profilers** (callgrind, dhat) record every event of interest. High overhead
  (10–100×), exact. Use when you specifically need exact counts, not when you need wall-clock.

Default to sampling. Switch to instrumentation only when sampling can't answer the question
(e.g., "how many bytes were allocated by this code path?" — sampling can't).

---

## 2. samply

Cross-platform sampling profiler that targets the Firefox Profiler UI. Easy to install, no
DTrace/perf permissions setup, generates a shareable URL.

### Install and use

```bash
cargo install samply

# Build with debug symbols in release
cargo build --profile release-with-debug    # see compiler-tuning.md

# Profile
samply record ./target/release-with-debug/your_binary <args>
```

Samply opens a local server, opens the Firefox Profiler in your default browser, and loads
the trace. Save with the share button to keep it.

### macOS notes

Samply uses macOS task_for_pid via a small embedded helper; on first run it prompts for
admin password to install. After that, no further setup. No DTrace tinkering needed.

### Linux notes

Samply uses `perf_event_open`. On most distros you'll need:

```bash
sudo sysctl kernel.perf_event_paranoid=-1
```

(or set it permanently in `/etc/sysctl.conf`).

### What samply gives you

The Firefox Profiler view shows:
- A flame graph (top of screen, time-ordered, zoomable)
- A call tree (left/right, inverted, etc.)
- Marker chart for spans/events if your code emits them
- Per-function breakdown

For most "where is time going?" questions, this is the fastest path to an answer.

---

## 3. cargo flamegraph

Wraps `perf` (Linux) or `dtrace` (macOS) and produces a static SVG flamegraph.

```bash
cargo install flamegraph

# Profile a binary
cargo flamegraph --bin my_app -- <args>

# Profile a benchmark
cargo flamegraph --bench my_bench -- --bench

# Profile a test
cargo flamegraph --test my_test -- <test_name>
```

Output: `flamegraph.svg` in the current directory. Open in a browser; click to zoom.

### macOS gotchas

- DTrace requires SIP-disabled or root. Most users run with `sudo cargo flamegraph`. Note that
  this changes target ownership of `target/`, which can cause subsequent non-sudo builds to
  fail. Workaround:
  ```bash
  sudo CARGO_HOME=$HOME/.cargo cargo flamegraph --bin my_app
  sudo chown -R $(whoami) target/
  ```
- For most macOS use, `samply` is less hassle.

### Linux gotchas

- Needs `perf_event_paranoid` lowered (see samply notes).
- For accurate stack traces, build with `[profile.release] debug = true` or use the
  `release-with-debug` profile.
- Frame pointers help. Force them with `RUSTFLAGS="-C force-frame-pointers=yes"` if call stacks
  look truncated.

---

## 4. Xcode Instruments

The macOS deep-analysis tool. Heavier than samply but unmatched for specific tasks:
- **Time Profiler** — sampling, like samply but with macOS's symbol resolution.
- **Allocations** — every malloc/free, with stack traces. Equivalent to dhat but lower friction.
- **System Trace** — syscalls, kernel transitions, scheduling.
- **Counters** — perf counters (cache misses, branch mispredicts). Apple Silicon exposes a
  curated set; Intel Macs expose more.
- **Metal** / **GPU Trace** — if you ever care.

### Workflow

1. Build with debug symbols: `cargo build --profile release-with-debug`.
2. Open Instruments (`open -a Instruments`).
3. Pick a template (Time Profiler is the usual start).
4. Set the target to your binary path; arguments under "Arguments".
5. Press the red record button. Stop when you have enough samples.
6. Heavy view: shows call tree by self-time and total-time. Right-click → "Charge..." to
   reattribute symbols you don't care about (e.g., libsystem_pthread).

Instruments is what to reach for when samply isn't enough — particularly for the Allocations
template, which is unique among free macOS tools.

---

## 5. perf (Linux)

The Linux gold standard. Available as `perf` or `linux-tools-common` package.

### Sampling

```bash
# Record
perf record --call-graph dwarf ./target/release-with-debug/my_app <args>

# Report (interactive TUI)
perf report

# Generate a flame graph
perf script | inferno-collapse-perf | inferno-flamegraph > flame.svg
```

`--call-graph dwarf` uses DWARF debug info for stack unwinding. Alternatives: `fp` (frame
pointers, faster but needs `force-frame-pointers`), `lbr` (last branch records, very fast on
modern Intel, limited stack depth).

### Counters

```bash
# Common counters
perf stat -e cycles,instructions,cache-misses,branch-misses ./bin

# Cache breakdown
perf stat -e L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./bin

# What events does my CPU support?
perf list
```

For "is this code cache-bound or compute-bound?" the answer is in `perf stat`. Rule of thumb:
IPC (instructions/cycle) below ~1 on modern x86 = stalled on something (cache, branch
mispredicts, dependency chains).

---

## 6. dhat (heap profiling, any OS)

A pure-Rust heap profiler. Add as a dev-dependency, wrap `main`, and you get a JSON dump that
loads into a viewer.

### Setup

```toml
[dev-dependencies]
dhat = "0.3"

[features]
dhat-heap = []
```

```rust
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;

fn main() {
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();

    // ... your code
}
```

Run with `cargo run --release --features dhat-heap`. On exit, dhat writes `dhat-heap.json`.
View at <https://nnethercote.github.io/dh_view/dh_view.html> (load the JSON file locally; no
upload to a server).

### What dhat shows

For each call stack:
- Total bytes allocated
- Maximum bytes alive at once
- Number of allocations
- Number of bytes freed before program end (vs. leaked)

This is the right tool when allocation pressure shows up in your sampling profile and you
need to know *which* code is doing it. It's not subject to the optimizer-defeats-microbench
problem because it instruments the global allocator directly.

### Performance impact

dhat slows the program down 2–5×. Fine for benchmarks; not for production. Compile with
`opt-level = 3` to keep it bearable.

---

## 7. heaptrack (Linux)

Heavier than dhat, with a GUI (`heaptrack_gui`) that visualizes allocations over time, leak
detection, and call-stack drill-down. Distro package: `heaptrack`.

```bash
heaptrack ./target/release-with-debug/my_app <args>
heaptrack_gui heaptrack.my_app.<pid>.zst
```

Use heaptrack when dhat's text-heavy view isn't cutting it for a large trace.

---

## 8. tokio-console

The right tool for "why is my async code slow?" or "is this task ever running?".

### Setup

```toml
[dependencies]
console-subscriber = "0.4"
tokio = { version = "1", features = ["full", "tracing"] }

# In your build:
# RUSTFLAGS="--cfg tokio_unstable"
```

```rust
fn main() {
    console_subscriber::init();
    // ... rest of main
}
```

### Use

```bash
cargo install tokio-console
# Run your app, then in another terminal:
tokio-console
```

You get a TUI with:
- All tasks, busy time, idle time, polls
- Resources (mutexes, semaphores, channels) and contention
- Warnings for tasks that block the runtime

Critical metrics:
- **Busy time per task** — high busy with low poll count = each poll does a lot (suspect:
  blocking work).
- **Idle time** — task is parked waiting for something. Cross-reference with the resource view.
- **Self-wakes** — task woke itself up. Often a sign of a busy-loop or spin.

---

## 9. cargo-asm / cargo-show-asm

When you need to know *what the compiler actually generated*. Both crates do similar work;
`cargo-show-asm` is the actively maintained one.

```bash
cargo install cargo-show-asm

cargo asm --release my_crate::my_function

# With Intel syntax
cargo asm --release --intel my_crate::my_function
```

Useful for verifying:
- A function was inlined (or wasn't).
- SIMD was actually generated (look for `vmulps`, `vaddps`, `vfmadd...`).
- Bounds checks were elided (no `panic_bounds_check` calls).
- A trait method was monomorphized (or boxed).

For vectorization specifically, the [Compiler Explorer](https://godbolt.org/) ("godbolt") with
the Rust compiler is often easier for short snippets.

---

## 10. Reading flamegraphs

Anatomy:
- **X-axis**: alphabetical, *not* time. Width = sample count = approximate time spent.
- **Y-axis**: stack depth. Bottom = `main`, top = leaf functions.
- **Width of a function** = total time including its callees.
- **Width above a function** = its callees' time.

What to look for, in order of usefulness:
1. **Wide leaves at the top** — that's where time is actually being spent.
2. **Wide internal nodes** — possible inlining failure, or an honest hot caller.
3. **Recurring patterns** — same function called from many places? Maybe one cold path is
   accidentally hot.

What to ignore:
- Narrow towers — they're noise unless you specifically care about p99 tails.
- Anything labeled `[unknown]` — you forgot debug symbols. Rebuild with debug = true and
  re-profile.

---

## 11. Common stack shapes and what they mean

| Profile shape | Likely cause | Action |
|--------------|--------------|--------|
| Wide `__rust_alloc` / `malloc` | Allocation pressure | See `allocations.md`. Reduce allocations, switch global allocator, use arenas. |
| Wide `memcpy` / `memmove` | Large value copies, possibly clones | Search for `.clone()`, `.to_vec()`, `Vec` resizes. Check for accidental owned-vs-borrowed. |
| Wide `core::fmt` / `format_args` | String formatting in hot path | Replace `format!` with `write!` to a reused buffer. |
| Wide hash table ops (`hashbrown::raw::*`) | Hashing dominates | Switch to `ahash`/`foldhash` if DoS not a concern. Check for needless rehashing. |
| Wide `spin_loop_hint` / `park` | Lock contention | See `async-tokio.md` (concurrency section). Consider sharding. |
| Wide `tokio::*::poll` with no leaf work | Async overhead exceeds work | Tasks too small; batch work or switch to sync. |
| Wide `_dl_runtime_resolve` / `__memcpy_*` | Dynamic linking overhead, glibc internals | Usually just glibc; rule out by relinking with musl or static. |
| Even spread, no peak | Cache or branch-bound | Run `perf stat` for IPC. Consider data layout, SoA, prefetching. |
| Wide `__rust_dealloc` | Drop overhead | Often a sign of large recursive structures (deep `Drop` chains). Consider arena allocation to bulk-free. |
