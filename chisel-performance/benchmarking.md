# Benchmarking Reference (Chisel-specific)

How to set up benchmarks that actually answer questions about Chisel performance.
This is layered on top of the general `rust-performance/references/benchmarking.md`;
read that first for criterion/divan/hyperfine basics.

## Table of contents

1. The two backends question
2. Per-layer microbenchmarks
3. End-to-end throughput benchmarks
4. Latency benchmarks
5. Comparison benchmarks (e.g., before/after a change)
6. CI-stable instruction-count benchmarks
7. Pitfalls specific to Chisel

---

## 1. The two backends question

Every Chisel benchmark must specify which backend it's measuring:

- **In-memory** (`Chisel::open_in_memory()`): no fsync, no filesystem, pure CPU/memory.
  Measures engine cost in isolation.
- **File** (`Chisel::open(path, opts)`): full path with fsync. Measures end-to-end
  including the storage layer.

A benchmark in just one backend tells you only half the story. Best practice: run the
same workload in both, report both. The ratio tells you where the cost lives.

```rust
fn bench_allocate(c: &mut Criterion) {
    let mut group = c.benchmark_group("allocate");

    group.bench_function("in_memory", |b| {
        let mut db = Chisel::open_in_memory().unwrap();
        b.iter(|| {
            db.begin().unwrap();
            db.allocate(black_box(b"hello")).unwrap();
            db.commit().unwrap();
        });
    });

    group.bench_function("file", |b| {
        let dir = tempfile::TempDir::new().unwrap();
        let path = dir.path().join("bench.db");
        let mut db = Chisel::open(&path, Options::default()).unwrap();
        b.iter(|| {
            db.begin().unwrap();
            db.allocate(black_box(b"hello")).unwrap();
            db.commit().unwrap();
        });
    });

    group.finish();
}
```

For the file backend, the `tempfile::TempDir` per-bench keeps tests isolated. For
sustained-throughput benches, a longer-lived file shows steady-state behavior better.

---

## 2. Per-layer microbenchmarks

The strict layer dependency makes per-layer benchmarking clean. Each layer's
benchmarks live in `benches/` named after the layer.

Suggested files:

- `benches/page_io.rs` — raw page reads/writes, with and without fsync.
- `benches/page_cache.rs` — cache hit/miss patterns, eviction cost.
- `benches/data_page.rs` — slot insert, slot lookup, compact cost.
- `benches/handle_table.rs` — lookup at various depths, COW path cost.
- `benches/transaction.rs` — begin/commit/rollback cost (in-memory).
- `benches/end_to_end.rs` — typical Chisel API workloads.

The advantage: when a profile flags a regression, you isolate which layer regressed
by running the matching bench file. Saves time vs. running everything.

### Example: handle table lookup at depth

```rust
fn bench_handle_lookup(c: &mut Criterion) {
    let mut group = c.benchmark_group("handle_table_lookup");

    for &handle_count in &[100usize, 100_000, 10_000_000] {
        // Populate a database with `handle_count` handles
        let mut db = Chisel::open_in_memory().unwrap();
        let mut handles = Vec::with_capacity(handle_count);
        db.begin().unwrap();
        for i in 0..handle_count {
            handles.push(db.allocate(format!("v{}", i).as_bytes()).unwrap());
        }
        db.commit().unwrap();

        group.bench_with_input(
            BenchmarkId::from_parameter(handle_count),
            &handles,
            |b, handles| {
                let mut rng = rand::thread_rng();
                b.iter(|| {
                    let h = handles[rng.gen_range(0..handles.len())];
                    db.read(black_box(h)).unwrap();
                });
            },
        );
    }
    group.finish();
}
```

This shows how lookup cost scales with tree depth — useful for catching regressions
that inadvertently slow the deep path.

---

## 3. End-to-end throughput benchmarks

For "is Chisel fast enough for my workload" questions, end-to-end benchmarks beat
microbenchmarks. Hyperfine works well here because the benchmark is "run the binary
that does the workload":

```bash
# bench_bulk_insert.rs is a binary that takes (count, batch_size, path) args
hyperfine --parameter-list batch_size 1,10,100,1000,10000 \
  './target/release/bench_bulk_insert 100000 {batch_size} /tmp/chisel-bench.db'
```

This produces the canonical "fsync amortization curve" — wall time as a function of
batch size — which is the single most useful thing to know about a Chisel workload.

For criterion equivalents, parameterized benchmarks with `BenchmarkId::from_parameter`
do the same job. Criterion is better for tracking regressions over time; hyperfine
is better for ad-hoc shape-of-the-curve questions.

---

## 4. Latency benchmarks

For workloads where p99 commit latency matters more than throughput:

```rust
use hdrhistogram::Histogram;

let mut hist = Histogram::<u64>::new(3).unwrap();
let mut db = Chisel::open(path, opts).unwrap();

for _ in 0..N {
    let start = Instant::now();
    db.begin().unwrap();
    db.allocate(b"x").unwrap();
    db.commit().unwrap();
    hist.record(start.elapsed().as_nanos() as u64).unwrap();
}

println!("p50: {} ns", hist.value_at_quantile(0.50));
println!("p99: {} ns", hist.value_at_quantile(0.99));
println!("p999: {} ns", hist.value_at_quantile(0.999));
println!("max: {} ns", hist.max());
```

`hdrhistogram` (the crate) gives you proper tail-aware percentiles. Criterion's mean +
stddev hides the tails.

For Chisel specifically, latency tails come from:

- Variable fsync time (kernel page cache state, drive write cache state).
- Defrag if it runs during measurement.
- HT depth changes (tree growth at threshold boundaries).
- File extension when the cache flushes a never-before-allocated page.

Identify the tail's source before "fixing" it; some of these are constants of the
storage layer, not Chisel issues.

---

## 5. Comparison benchmarks

When evaluating a proposed optimization:

1. Establish a baseline: `cargo bench --bench all -- --save-baseline main`
2. Apply the change.
3. Compare: `cargo bench --bench all -- --baseline main`

Criterion reports percentage change with statistical significance. Anything within
the noise band (criterion's default 5%) is not a real change.

For Chisel: **always run both backends**. A change that improves in-memory by 20%
but regresses file by 5% is usually a bad trade — the file backend is what users
see.

### The "is the change durable" test

After establishing improved numbers, the optimization PR should include:

- A test in the relevant module that exercises the change's correctness boundary.
- A bench reproducing the improvement.
- A note in `ISSUES.md` documenting the change with date stamp and rationale.

This is part of the project's standard PR practice; not specific to performance.

---

## 6. CI-stable instruction-count benchmarks

GitHub Actions runners have noisy wall-clock measurements. For CI regression detection
that survives runner noise, use `iai` or `iai-callgrind`:

```toml
[dev-dependencies]
iai = "0.1"
# or
iai-callgrind = "0.10"
```

```rust
use iai::black_box;

fn bench_allocate_inline() -> u64 {
    let mut db = Chisel::open_in_memory().unwrap();
    db.begin().unwrap();
    let h = db.allocate(black_box(b"hello world")).unwrap();
    db.commit().unwrap();
    h
}

iai::main!(bench_allocate_inline);
```

`iai` measures via callgrind: instruction count, L1/LL cache reads/writes, branch
mispredicts. These are deterministic across runs and machines. Trade-off: no actual
wall-clock numbers, no fsync (callgrind makes I/O too slow to be representative).

So CI uses `iai` for regression detection (any 5% increase in instruction count is a
flag), and local development uses criterion for actual wall-clock numbers.

---

## 7. Pitfalls specific to Chisel

### Filesystem cache effects

If you run a file-backed bench in a tight loop, the OS page cache warms up and fsync
becomes much faster than first-write fsync. To measure cold-cache fsync:

```bash
# Linux
sudo sysctl -w vm.drop_caches=3

# macOS
sudo purge
```

Run between bench runs that need cold-cache realism. Or use a fresh tempdir for each
sample (criterion does this if you set up the bench correctly).

### Filesystem differences

- **APFS (macOS)** has different fsync semantics than ext4 — generally faster, but
  the durability guarantees differ.
- **ext4 on a typical Linux dev box** is the most "expected" behavior; tune
  benchmarks to ext4 baseline.
- **tmpfs** is essentially in-memory but goes through the syscall path. A useful
  middle ground for "what if storage were free?" measurements without re-coding.
- **Network filesystems** (NFS, SMB) have wildly variable fsync. Avoid for benching;
  the noise drowns the signal.

If a benchmark looks suspicious, the first question is "what filesystem?".

### Drive write cache

Many consumer SSDs lie about fsync — they return success when the write hit the
drive's volatile cache, not non-volatile storage. Chisel measurements on such drives
overestimate durability throughput. For real durability numbers, use:

- macOS: `F_FULLFSYNC` (not currently exposed by Chisel — would need an opt-in API).
- Linux: drives with power-loss protection (datacenter-grade), or disable write cache
  via `hdparm -W 0`.

For benchmarking purposes, knowing the drive's behavior matters more than its
absolute speed.

### CPU frequency scaling

For comparison benchmarks, pin the CPU governor to performance:

```bash
# Linux
sudo cpupower frequency-set --governor performance

# Restore
sudo cpupower frequency-set --governor schedutil
```

On macOS this is largely automatic but Time Profiler in Instruments has a
"System Trace" mode that shows the actual frequencies during the run.

### `release-with-debug` for profiling

Always profile a build with debug symbols, never the default release. Add to
`Cargo.toml`:

```toml
[profile.release-with-debug]
inherits = "release"
debug = "line-tables-only"
strip = "none"
```

Then: `cargo build --profile release-with-debug && samply record ./target/release-with-debug/...`

### `target-cpu=native` for the benchmark machine

For benchmarks that will be compared against themselves on the same machine:

```bash
RUSTFLAGS="-C target-cpu=native" cargo bench
```

This enables AVX2 / NEON / etc. for the build CPU. The numbers won't be portable to
other CPUs, but they're realistic for "what the engine could do on this hardware."
