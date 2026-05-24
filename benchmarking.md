# Benchmarking Reference

This file covers the three benchmarking tools you should know, the methodology pitfalls that
make Rust microbenchmarks lie, and how to set up each tool for a new project.

## Table of contents

1. Criterion — established, statistical, slow
2. Divan — newer, faster, simpler
3. Hyperfine — whole-program / CLI timing
4. Statistical methodology
5. The `black_box` trap
6. Throughput vs latency benches
7. Comparing benchmarks across runs
8. CI integration

---

## 1. Criterion

The de facto standard for Rust microbenchmarks. Statistically rigorous (gives confidence
intervals, detects regressions across runs, plots distributions), but slow to compile and
slow to run.

### Setup

```toml
# Cargo.toml
[dev-dependencies]
criterion = { version = "0.5", features = ["html_reports"] }

[[bench]]
name = "my_bench"
harness = false  # disable libtest harness; criterion provides its own
```

```rust
// benches/my_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_fib(c: &mut Criterion) {
    c.bench_function("fib 20", |b| {
        b.iter(|| fib(black_box(20)))
    });
}

criterion_group!(benches, bench_fib);
criterion_main!(benches);
```

Run with `cargo bench --bench my_bench`. Criterion writes HTML reports to
`target/criterion/<name>/report/index.html`.

### Useful patterns

**Parameterized benchmarks** — measure across input sizes:

```rust
fn bench_parse(c: &mut Criterion) {
    let mut group = c.benchmark_group("parse");
    for size in [10, 100, 1_000, 10_000].iter() {
        let input = generate_input(*size);
        group.throughput(Throughput::Bytes(input.len() as u64));
        group.bench_with_input(BenchmarkId::from_parameter(size), &input, |b, input| {
            b.iter(|| parse(black_box(input)))
        });
    }
    group.finish();
}
```

`Throughput::Bytes` lets criterion report MB/s. `Throughput::Elements` for ops/sec.

**Benchmarks that need setup** — use `iter_with_setup` or `iter_batched`:

```rust
b.iter_batched(
    || expensive_setup(),         // setup, not measured
    |input| process(input),       // measured
    BatchSize::SmallInput,
);
```

### Why criterion is slow

- It compiles a separate binary per `[[bench]]` entry, with all dev-dependencies.
- It runs each benchmark for ~3 seconds of warmup + ~5 seconds of measurement by default.
- HTML report generation adds time.

For day-to-day iteration on a single benchmark, prefer `divan`.

---

## 2. Divan

Newer, simpler, much faster to compile and run. Inspired by Google Benchmark. Less statistical
machinery, no HTML reports, but fine for the iteration loop.

### Setup

```toml
[dev-dependencies]
divan = "0.1"

[[bench]]
name = "my_bench"
harness = false
```

```rust
// benches/my_bench.rs
fn main() {
    divan::main();
}

#[divan::bench]
fn fib_20() -> u64 {
    fib(divan::black_box(20))
}

#[divan::bench(args = [10, 100, 1_000, 10_000])]
fn parse(bencher: divan::Bencher, size: usize) {
    let input = generate_input(size);
    bencher.bench_local(|| parse(divan::black_box(&input)));
}
```

Divan's output is a clean table to stdout. No HTML, no crates.io dependencies pulled in for
plotting. Compile time is dramatically lower.

### When to use which

- **Fast iteration during development** → divan.
- **CI regression detection / publishing benchmark results** → criterion.
- Many real projects use both: divan for the inner loop, criterion for the published, archived
  numbers.

---

## 3. Hyperfine

Wall-clock timing of arbitrary commands. Use for CLI tools, end-to-end timing, or when you
want to compare a Rust binary against alternatives in another language.

```bash
cargo install hyperfine

# Compare two binaries
hyperfine --warmup 3 './target/release/old' './target/release/new'

# With parameters
hyperfine --parameter-scan size 100 10000 -D 100 \
  './target/release/parse --size {size}'

# Export results
hyperfine --export-json results.json --export-markdown results.md './bin'
```

Hyperfine handles: warmup runs, statistical outlier detection, shell overhead measurement,
and color-coded comparison. It's the right tool when "the benchmark" is invoking a binary,
not calling a function.

---

## 4. Statistical methodology

Microbenchmarks lie if you let them. The defenses:

- **Run on idle hardware.** Close everything. CPU frequency scaling, thermal throttling, and
  background processes will dominate small differences. On macOS, disable Spotlight indexing
  during benchmarking.
- **Disable CPU frequency scaling on Linux.**
  ```bash
  sudo cpupower frequency-set --governor performance
  ```
  Restore to `powersave` or `schedutil` after.
- **Pin to a core** to avoid migration. Linux: `taskset -c 0 cargo bench`. Useful when comparing
  small differences.
- **Account for variance.** Criterion does this; if you're rolling your own, run at least 30
  iterations and report median + IQR, not mean.
- **Beware "improvement" smaller than the run-to-run variance.** A 2% speedup on a benchmark
  with 5% variance is noise. Use criterion's regression detection, or run before/after pairs
  multiple times and check the distributions overlap or not.
- **Beware compiler caching.** A second run after a code change may pick up cached compilation
  artifacts that look faster. `cargo clean` between architectural changes if in doubt.

---

## 5. The `black_box` trap

LLVM is very good at optimizing away work whose result isn't observed. A naive microbenchmark:

```rust
b.iter(|| fib(20))   // BAD: result unused, may be constant-folded
```

The fix:

```rust
b.iter(|| fib(black_box(20)))    // input is opaque to optimizer
b.iter(|| black_box(fib(20)))    // output is observed
```

Both are needed in some cases — the input `black_box` prevents constant folding of the
argument, the output `black_box` prevents dead-code elimination of the result. Criterion's
`b.iter` provides some of this implicitly, but explicit `black_box` is safer.

`std::hint::black_box` is the stable version. Criterion and divan re-export their own; use
those inside their benchmark functions.

### Telltale signs your bench is being optimized away

- Time per iteration is suspiciously low (<1 ns).
- Doubling the input size doesn't change the time.
- The benchmark for a complex function is faster than the benchmark for a simple function.

If any of these is true, audit the `black_box` placement and inspect the generated assembly
with `cargo asm` or `cargo show-asm`.

---

## 6. Throughput vs latency benches

These are different measurements and need different setups.

**Throughput** — how much work per unit time? Use `Throughput::Bytes` or `Throughput::Elements`,
report ops/sec or MB/s. Batch the work inside `iter` for better signal:

```rust
b.iter(|| {
    for item in &batch {
        process(black_box(item));
    }
});
```

**Latency** — how long does one operation take? Single operation per iteration, no batching.
Critical for tail-latency-sensitive systems:

```rust
b.iter(|| process(black_box(&single_item)));
```

For tail latency specifically, criterion's distribution view (HTML report) is essential — mean
or median can hide a long tail. For real production p99 measurement, you need histograms in
the application itself (`hdrhistogram` crate), not benchmarks.

---

## 7. Comparing benchmarks across runs

Criterion stores baselines automatically. To compare:

```bash
# Save current as baseline named "main"
cargo bench --bench my_bench -- --save-baseline main

# After making changes, compare
cargo bench --bench my_bench -- --baseline main
```

Output shows percentage change with statistical significance. This is the right way to verify
an optimization actually worked.

For divan, comparison is manual — run before, copy the output, run after, diff.

---

## 8. CI integration

Two main approaches:

1. **`bencher.dev`** or **`codspeed.io`** — hosted services that run criterion benches on every
   PR and detect regressions. Free tiers exist.
2. **Self-hosted** — store baseline JSON in the repo, compare on PR. Fragile due to noisy
   GitHub Actions runners. Codspeed's trick is to use Valgrind/cachegrind for instruction
   counts (deterministic) instead of wall clock.

For instruction-count-based benchmarking that survives noisy CI:
- `iai` crate — wraps cachegrind, counts instructions executed
- `iai-callgrind` — successor, more features

These trade physical realism (cache effects, branch prediction) for repeatability. Useful as a
regression gate; not a substitute for wall-clock benches before release.
