# Async & Tokio Performance Reference

Async code has its own performance model. The CPU-time view of a profiler shows you what your
code is doing while running, but a lot of async pathology is about *not* running — tasks
parked on locks, futures held over awaits, runtime overhead exceeding work.

## Table of contents

1. The mental model: futures and the runtime
2. Choosing a runtime flavor
3. Worker thread tuning
4. The blocking pool
5. tokio-console — the diagnostic tool
6. Lock contention in async code
7. Channel selection
8. Cancellation and timeouts
9. async-trait alternatives
10. When to *not* use async

---

## 1. The mental model

A Rust async function compiles to a state machine. Each `.await` is a state transition: the
function may yield control back to the runtime, which polls other tasks, then resumes when
the awaited operation is ready.

The runtime (Tokio, in 99% of cases) is a multi-threaded executor with:
- A pool of **worker threads** that run async tasks.
- A separate pool of **blocking threads** for synchronous work.
- An I/O reactor (epoll/kqueue/IOCP) that wakes tasks when their I/O is ready.

Performance pathologies in this model:
- A task that does CPU-heavy work without yielding blocks its worker thread, starving other
  tasks.
- A task that holds a lock across `.await` blocks every other task that wants the lock.
- A task that does sync I/O on a worker thread blocks the worker.
- Too many tasks for the runtime to schedule fairly causes wakeup latency spikes.
- Too few worker threads under-uses CPU.
- Too many worker threads thrash the cache.

---

## 2. Choosing a runtime flavor

```rust
// Multi-threaded (default)
#[tokio::main]
async fn main() { ... }

// Single-threaded
#[tokio::main(flavor = "current_thread")]
async fn main() { ... }
```

| Flavor | Best for | Worst for |
|--------|---------|-----------|
| Multi-threaded | General I/O-heavy services, mixed workloads | Workloads where work-stealing overhead exceeds the work |
| Single-threaded (`current_thread`) | Per-core sharded designs, small/embedded, deterministic ordering | Anything that needs to use multiple CPU cores in one runtime |

A common pattern for high-performance services: spawn N single-threaded runtimes, one per
core, with each handling its own sharded subset of work. Eliminates work-stealing and lock
overhead entirely.

---

## 3. Worker thread tuning

Default: one worker per CPU core. Override:

```rust
let runtime = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(8)
    .thread_name("my-worker")
    .enable_all()
    .build()
    .unwrap();
```

When to tune:
- **Reduce** worker count if your workload is mostly I/O-bound and tasks are short. Default
  N-cores can over-thread for purely-I/O work.
- **Increase** if you have a hybrid workload that's both I/O-heavy and CPU-heavy (rare; usually
  better to separate the CPU work via `spawn_blocking` or a separate runtime).
- For NUMA machines, consider one runtime per NUMA node with thread pinning. Tokio doesn't do
  this for you; libraries like `glommio` (Linux io_uring) are designed around it.

---

## 4. The blocking pool

Tokio has a separate pool for blocking work, accessed via `spawn_blocking`. Default size: 512
threads. The number is high so that occasional blocking work doesn't stall, but it's almost
always too high for a real workload.

```rust
let runtime = tokio::runtime::Builder::new_multi_thread()
    .worker_threads(8)
    .max_blocking_threads(32)
    .build()?;
```

Set this to a number that reflects your actual sync I/O concurrency. 512 threads × 2 MB stack
= 1 GB of address space reserved for blocking work, even if you never use it.

### When to use `spawn_blocking`

- Sync filesystem I/O.
- Sync DB drivers (e.g., the `rusqlite` driver if you must use it).
- CPU-heavy computation that would otherwise block a worker for >100 µs.
- Any FFI call that blocks.

Don't use it for:
- Trivial work (the spawn cost exceeds the savings).
- Work that itself uses async — it'll create a runtime-in-a-thread tangle.

---

## 5. tokio-console

The single most valuable tool for diagnosing async pathology. It's TUI-based (no GUI required)
and shows you the runtime's view of every task.

### Setup

```toml
[dependencies]
console-subscriber = "0.4"
tokio = { version = "1", features = ["full", "tracing"] }
```

```rust
fn main() {
    console_subscriber::init();
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async_main());
}
```

Build with:

```bash
RUSTFLAGS="--cfg tokio_unstable" cargo build --release
```

(Tokio's tracing instrumentation is gated behind the `tokio_unstable` cfg flag — this is
intentional, the tracing API is not yet stable.)

### Use

```bash
cargo install tokio-console

# Run your app, then:
tokio-console
```

### What to look for

- **Tasks with high "busy" time and low poll counts**: each poll does a lot of work. Possible
  blocking work happening; consider `spawn_blocking`.
- **Tasks with many self-wakes**: the task wakes itself repeatedly. Often a busy-wait pattern
  or a poorly-written future.
- **Resources with high contention**: a `Mutex` or `Semaphore` that many tasks queue on. This
  is your serialization point.
- **Long-poll warnings** (red highlighting): tokio detected a poll that took too long, blocking
  the worker. Investigate immediately.

The single most common finding: a sync operation accidentally on the async runtime (e.g.,
`std::fs::File::open` instead of `tokio::fs::File::open`).

---

## 6. Lock contention in async code

The most pernicious async bug:

```rust
// BAD: lock held across .await
async fn process(state: Arc<Mutex<State>>) {
    let guard = state.lock().unwrap();   // std Mutex
    do_async_work().await;               // guard held, runtime serializes
    // ...
}
```

If `do_async_work` takes 10 ms, every other task wanting `state` waits 10 ms. The runtime is
fine; your throughput is destroyed.

Worse: with `std::sync::Mutex`, this can deadlock — if `do_async_work` schedules to the same
thread, you can re-enter the lock-holder.

### The fixes

1. **Restructure** to release the lock before awaiting:
   ```rust
   let snapshot = {
       let guard = state.lock().unwrap();
       guard.snapshot()    // cheap clone of relevant data
   };
   do_async_work_with(snapshot).await;
   ```

2. **Use `tokio::sync::Mutex`** if you genuinely need to hold a lock across await:
   ```rust
   let guard = state.lock().await;   // tokio Mutex; safe across await
   do_async_work().await;
   ```
   Slower than `std::Mutex` per-acquisition; safe for cross-await use.

3. **Use a different concurrency primitive**:
   - `arc_swap::ArcSwap` for read-mostly data — readers are lock-free.
   - `dashmap::DashMap` for concurrent maps — sharded internally.
   - `papaya::HashMap` — newer lock-free map, very fast reads.
   - Per-task ownership via `Actor` patterns (each task owns its state, communicates via
     channels).

### How to spot it

- `tokio-console` will show high contention on the resource.
- Profiling will show wide `parking` / `condvar_wait` stacks.
- The classic symptom: throughput collapses non-linearly as concurrent users rise.

---

## 7. Channel selection

Tokio gives you several channel types. Pick wrong and you pay.

| Channel | When |
|---------|------|
| `tokio::sync::mpsc` | Many producers, one consumer. The default. Bounded variant for backpressure. |
| `tokio::sync::oneshot` | Single message, single consumer. Lighter than mpsc. |
| `tokio::sync::broadcast` | One producer, many consumers. Each consumer sees every message. |
| `tokio::sync::watch` | Latest-value semantics. Late subscribers see only the most recent value. |
| `flume` | Drop-in mpsc with sync-or-async API; often faster than tokio's mpsc. |
| `crossbeam::channel` | Sync only; lower overhead than tokio's mpsc when you don't need async. |

Bounded vs unbounded: bounded for backpressure (slow consumer slows producer), unbounded for
fire-and-forget. Unbounded under load can grow without limit and OOM the process; prefer
bounded with a reasonable limit unless you have specific reason.

---

## 8. Cancellation and timeouts

Async cancellation is structured: dropping a future cancels it. This is mostly free. But it
has subtle costs:

- **Drop chains** — cancelling a future drops everything it owns. If those owns are large or
  have non-trivial drop, the cancellation is not free.
- **Resource cleanup** — a cancelled task doesn't get a chance to run cleanup code. If you
  hold an `acquire`d resource (semaphore, lock, file lock), use RAII or explicit cleanup.

`tokio::time::timeout` is the standard way to bound work:

```rust
match tokio::time::timeout(Duration::from_secs(5), do_work()).await {
    Ok(result) => result,
    Err(_elapsed) => return Err("timed out"),
}
```

Be aware: the timer wheel inside tokio has bounded resolution (~1 ms by default). Sub-ms
timeouts work but can be imprecise.

---

## 9. `async-trait` alternatives

`async fn` in traits has been a long-running pain point. As of recent stable releases, you
can write:

```rust
trait Service {
    async fn handle(&self, req: Request) -> Response;
}
```

But this has caveats around `Send` bounds and dynamic dispatch. The current state:

- For trait objects (`dyn Trait`), you still need either:
  - The `async-trait` crate (boxes the future, allocation per call).
  - The `trait-variant` crate (Send variants).
- For static dispatch, `async fn` in trait is stable and zero-cost.

For hot-path traits, prefer static dispatch (`impl Service` rather than `dyn Service`) or
manually-written `Future`-returning methods. The `async-trait` macro is convenient but the
per-call allocation matters in hot paths.

---

## 10. When to *not* use async

Async is the right answer for I/O-bound concurrency at scale. It's the wrong answer when:

- **The work is CPU-bound.** Use `rayon` for parallel data work, `std::thread::spawn` for
  one-off background tasks.
- **You have only a few concurrent operations.** The thread overhead of OS threads (~1 MB
  stack) is fine for tens of concurrent operations; the overhead of async machinery may not be
  worth it.
- **You need predictable latency more than throughput.** Async runtimes have head-of-line
  blocking when a task misbehaves. A thread-per-connection model is more isolated.
- **The library you depend on is sync only.** Wrapping sync calls in `spawn_blocking` works,
  but if every call goes through the blocking pool, you're paying async overhead for no
  concurrency benefit.

Async vs. threads is not a one-way door. Some real-world systems use both: an async front-end
for I/O concurrency, dispatching CPU work to a `rayon` thread pool.
