# Python Binding Performance Reference

The `chisel-py` PyO3 binding adds overhead distinct from the Rust core. This file
covers the costs, the optimizations available, and how to measure both.

## Table of contents

1. The PyO3 cost stack
2. The GIL and `allow_threads`
3. Value marshaling: `Vec<u8>` ↔ `PyBytes`
4. Per-call overhead and bulk methods
5. Async Python and Chisel
6. Benchmarking the binding
7. When to push work into Rust vs Python

---

## 1. The PyO3 cost stack

A Chisel call from Python pays:

```
Python call            (~0.1 µs typical Python function call)
  ↓
PyO3 boundary entry    (~0.5–1 µs: argument extraction, GIL state)
  ↓
Rust Chisel call       (the engine work)
  ↓
Result conversion      (~0.5–2 µs: PyBytes alloc, type wrapping)
  ↓
PyO3 boundary exit     (~0.5 µs)
  ↓
Python returns         (~0.1 µs)
```

For a cache-hit `read()` that takes ~1 µs in Rust, the PyO3 stack adds 1–4 µs —
the binding overhead can dominate the engine cost. For a `commit()` that takes
hundreds of microseconds, it's negligible.

This is the central tension: **Python-side optimization matters more for fast
operations than for slow ones.**

---

## 2. The GIL and `allow_threads`

By default, a PyO3 function holds the GIL for its entire duration. This means:

- Other Python threads in the same process cannot run during the call.
- For a long Chisel operation (commit, defrag, large overflow read), the GIL
  is held for milliseconds.

For long operations, release the GIL inside the binding:

```rust
#[pyfunction]
fn commit(py: Python<'_>, db: &mut PyChisel) -> PyResult<()> {
    py.allow_threads(|| db.inner.commit())
        .map_err(into_py_err)
}
```

`allow_threads` releases the GIL for the closure's duration. Other Python threads
can run; Rust code inside cannot touch any Python objects (the borrow checker
enforces this).

### When to release the GIL

| Operation | Typical duration | GIL release? |
|-----------|------------------|--------------|
| `read()` (cache hit) | ~1–5 µs | Don't bother — boundary cost exceeds work |
| `read()` (cache miss + small value) | ~50 µs–1 ms | Yes — there's I/O |
| `read()` (large overflow) | ms range | Yes |
| `allocate()` (small value) | ~1–10 µs | Don't bother |
| `update()` | similar to allocate | Don't bother |
| `commit()` | ~100 µs–10 ms | **Always** — the fsyncs are the longest hold |
| `defrag()` | seconds possible | **Always** |
| `delete_many()` (large batch) | ms range | Yes |

The rule of thumb: release GIL for any operation likely to exceed ~10 µs.

### Verifying GIL release in chisel-py

When adding a new binding method, the test should verify GIL release. A way to do
this:

```python
import threading
import time

def test_commit_releases_gil(db):
    counter = {"value": 0}

    def background():
        for _ in range(1000):
            counter["value"] += 1
            time.sleep(0.0001)

    t = threading.Thread(target=background)
    t.start()
    db.begin()
    for i in range(10000):
        db.allocate(b"test value")
    db.commit()   # if GIL is released, counter advances during this call
    t.join()

    assert counter["value"] > 100   # background made progress
```

---

## 3. Value marshaling

Every `read()` from Python:

1. Rust reads into `Vec<u8>`.
2. PyO3 wraps it as `PyBytes` (allocates a new Python `bytes` object, copies the data).

For small values, the copy is fast (memcpy of <1 KB). For large overflow values, it's
a significant cost — and *necessary*, because `PyBytes` owns its data.

Alternative: `PyByteArray` (mutable, similar cost) or `memoryview` over an
`Arc`-shared buffer (zero-copy, complex lifetime management).

### When to optimize value marshaling

For a workload that reads many large values into Python, the PyBytes copy can
dominate. Profile with `cProfile` or `py-spy` first; if the binding shows up as the
hot spot, consider:

- Returning `bytearray` if the caller writes to it (bytes are immutable in Python;
  bytearray isn't, and skipping a "copy on first write" can help).
- A streaming read API: `read_into(handle, buffer: bytearray)` writes into a
  caller-provided buffer with no allocation.
- A Python-side cache for hot reads.

For the common case (small-to-medium values, occasional reads), PyBytes is fine.

### Allocation patterns at the binding

Python's bytes pool is good for short strings but every read still allocates. For
high-throughput reads, the allocator matters: `mimalloc` (Python build) or simply
ensuring Python's GC isn't triggering during the bench.

---

## 4. Per-call overhead and bulk methods

The 1–4 µs PyO3 boundary cost is per call. For workloads doing thousands of small
operations from Python, the boundary cost can exceed Chisel's engine cost.

The fix: bulk methods at the Python boundary.

The Rust core has `delete_many(handles: &[u64])`. Equivalent bulk wrappers worth
adding to chisel-py:

- `allocate_many(values: list[bytes]) -> list[int]`
- `read_many(handles: list[int]) -> list[bytes]`
- `update_many(pairs: list[tuple[int, bytes]]) -> None`
- `delete_many(handles: list[int]) -> None`  (already in Rust core)

These cross the PyO3 boundary once, do N operations in Rust, return one result.
The savings:

```
N separate calls:    N × (boundary cost + engine work)
1 bulk call:         1 × boundary cost + N × engine work
                     ~ N × engine work
```

For N = 1000, small operations, this is 1000 × 4 µs = 4 ms saved on the boundary
alone — substantial for tight read loops.

### When to add bulk methods

Add a bulk method when:
- The corresponding non-bulk operation has high call-to-work ratio (boundary cost ≥
  engine cost).
- A real workload pattern matches: ETL pipelines, batch processors, bulk loads.

Don't add bulk methods speculatively; each one is API surface.

---

## 5. Async Python and Chisel

Python's `asyncio` and Chisel's synchronous `&mut self` API have a tension. Two
patterns work:

### Pattern 1: `asyncio.to_thread` per call

```python
async def get(handle: int) -> bytes:
    return await asyncio.to_thread(db.read, handle)
```

Simple. Each call goes to a thread pool, which acquires the GIL only when needed.
The Chisel call runs in the worker thread; the `read()` releases the GIL during its
fsync (if applicable), so other workers and the asyncio loop can run.

Cost: thread pool dispatch overhead (~10 µs). Fine for "bigger" operations
(`commit`, large `read`). Wasteful for small ops.

### Pattern 2: A dedicated worker thread

```python
class ChiselWorker:
    def __init__(self, db):
        self.db = db
        self.queue = asyncio.Queue()
        self.thread = threading.Thread(target=self._run, daemon=True)
        self.thread.start()

    def _run(self):
        loop = asyncio.new_event_loop()
        while True:
            op, fut = loop.run_until_complete(self.queue.get())
            try:
                fut.set_result(op(self.db))
            except Exception as e:
                fut.set_exception(e)

    async def call(self, op):
        fut = asyncio.get_event_loop().create_future()
        await self.queue.put((op, fut))
        return await fut
```

All Chisel access goes through one thread. The asyncio loop is unblocked. Useful
when you want strict single-writer semantics from many async tasks.

Cost: serialization through one thread (no parallelism). Acceptable because Chisel
is single-writer anyway.

### Anti-pattern: calling Chisel from the asyncio loop directly

```python
# BAD
async def bad(handle):
    return db.read(handle)   # blocks the loop on fsync if commit is happening
```

Even though `read()` is fast on cache hits, anything reaching disk blocks the entire
loop. Always go through `asyncio.to_thread` or a worker thread.

---

## 6. Benchmarking the binding

Two distinct benches:

### Boundary-cost bench

Measures PyO3 overhead in isolation:

```python
import chisel
import time

db = chisel.open(None)   # in-memory mode
db.begin()
h = db.allocate(b"x")
db.commit()

start = time.perf_counter_ns()
for _ in range(100_000):
    db.read(h)   # cache hit, minimal engine work
elapsed = time.perf_counter_ns() - start

per_call_ns = elapsed / 100_000
print(f"per-call: {per_call_ns:.0f} ns")
```

Compare against the Rust microbench for the same operation. The difference is the
PyO3 boundary cost.

### End-to-end bench

Use `pytest-benchmark` for in-test benchmarks, or just a Python script:

```python
def bench_bulk_insert(db, n: int, batch: int):
    start = time.perf_counter()
    for chunk_start in range(0, n, batch):
        db.begin()
        for i in range(chunk_start, min(chunk_start + batch, n)):
            db.allocate(f"value-{i}".encode())
        db.commit()
    return time.perf_counter() - start
```

Compare across batch sizes; the curve mirrors the Rust-side fsync amortization curve
but adds the per-call boundary cost.

---

## 7. When to push work into Rust vs Python

Decision rule: if the operation is "loop calling Chisel methods," push it to Rust.

Signs work belongs in Rust:
- Tight loop over many handles.
- Per-element transformation that's defined in code, not data.
- Performance-critical, profiled as boundary-cost-dominated.

Signs work can stay in Python:
- One-off operations (open, close, occasional reads).
- Logic that interacts heavily with Python data structures.
- Code that changes frequently (Rust requires a rebuild).

The chisel-py module is small. Be conservative about adding methods — each one is API
surface, version-bound. A handful of well-chosen bulk operations covers most needs.
