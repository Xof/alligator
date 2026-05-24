---
name: chisel-performance
description: >
  Performance work specific to Chisel — the transactional shadow-paging single-writer
  Rust storage engine and its PyO3 binding (chisel-py). Use proactively whenever
  working in the Chisel codebase or reviewing PRs that touch performance-sensitive
  paths. Trigger on Chisel module names (page_cache, handle_table, transaction,
  page_io, data_page, freemap, overflow, defrag, superblock), Chisel concepts (shadow
  paging, COW pages, slot packing, handle table radix tree, fsync, poison model,
  in-memory mode, named roots, savepoints, watermark rollback, format_version), or
  Chisel-specific concerns (commit slow, fsync overhead, cache hit rate, transaction
  batching, CacheFull, XXH3 cost, defrag scheduling, write amplification). Always
  treat durability, poison-model integrity, on-disk format stability, fsync ordering,
  and the single-writer &mut self contract as hard constraints — Chisel is
  durability-first. Defer to rust-performance for general Rust patterns.
---

# Chisel Performance Skill

Audience: contributors working on Chisel (the storage engine), `chisel-py` (the PyO3
binding), or downstream code calling Chisel directly. Assumes familiarity with Chisel's
architecture (`ARCHITECTURE.md`); this skill won't re-explain shadow paging.

## The First Principle

**Durability is the priority. Performance is what we optimize within the durability
envelope, never against it.** Many fast designs are unavailable to Chisel by construction:

- No batched/group commit across processes — Chisel is single-writer.
- No async I/O — `&mut self` and synchronous fsync are part of the contract.
- No relaxed fsync — fsyncgate semantics make retry unsafe; the poison model is
  non-negotiable.
- No format-breaking optimization — within a major, every byte on disk is sacred.
- No reordering of the commit protocol — the data-fsync-before-superblock-fsync invariant
  is what makes shadow paging work.

If a proposed optimization touches any of the above, it is wrong even if it benchmarks
faster. See [Don't-Break List](#dont-break-list).

For general Rust performance technique (profiling tools, allocator selection, criterion
setup, antipattern catalog), use the `rust-performance` skill. This skill is for things
that are specific to Chisel's architecture.

---

## The Cost Model

Optimization without a cost model leads to local wins that don't add up. For Chisel,
every operation has a small, predictable cost profile.

### Per-operation costs

| Operation | Page reads (cold) | Page reads (cached) | Pages dirtied | Allocations | Notes |
|-----------|-------------------|---------------------|---------------|-------------|-------|
| `read(h)` (inline) | 1–3 (HT) + 1 (data) | 0 | 0 | 1 `Vec<u8>` | HT depth = log_1021(handle_count) |
| `read(h)` (overflow) | 1–3 (HT) + N (chain) | 0–N | 0 | 1 `Vec<u8>` | N = chain length |
| `allocate(v)` (inline) | 1–3 (HT path, may COW) | 0 | 1 data page (often shared) + 1–3 HT COW | new pages | Slot packing reuses a page |
| `allocate(v)` (overflow) | 1–3 (HT) | 0 | N+1 overflow + 1 HT | new pages | N = ceil(len / OVERFLOW_PAYLOAD) |
| `update(h, v)` (same size) | as `read` | 0 | 1 data page (COW) + 1–3 HT COW | new pages | |
| `update(h, v)` (grew) | as `read` | 0 | new pages for value + HT COW | new pages | Old pages free at commit |
| `delete(h)` | 1–3 (HT) | 0 | 1–3 HT COW | 0 | Tombstone written to leaf |
| `commit()` | 0 | 0 | 1 freemap (new) + 1 superblock | freemap COW | **2 fsyncs minimum, 3 with pre-drain** |
| `rollback()` | 0 | 0 | 0 (drop dirty) | 0 | Free; pages above watermark drop |
| `savepoint(name)` | 0 | 0 | 0 | bookkeeping | Lightweight stack push |
| `defrag(opts)` | many | varies | many (rewrite) | many | Schedule explicitly |

### The fsync floor

Every commit performs **2 fsyncs** at minimum (data, then superblock), and **3 fsyncs**
in the common path because the pre-drain flush in `TransactionManager::commit` adds one
to keep `persist_freemap` from tripping `CacheFull` mid-commit.

On a typical NVMe SSD, an fsync is on the order of 100 µs to several ms; on rotational
storage, 5–20 ms. **This is the dominant cost of any committing workload.** Most Chisel
optimization lives in the question "how do I do more work per commit?" — see
[Lever 1: Transaction Batching](#lever-1-transaction-batching).

### Where time goes inside a commit (not counting fsync)

For commits with M dirty pages and H affected handles:

- Handle table COW: O(H × tree_depth) page writes, O(H) checksum computes.
- Freemap COW: 1 page write, 1 checksum.
- Data/overflow page writes: M page writes, M checksum computes.
- Superblock write: 1 page write, 1 checksum.
- Total CPU time scales with M; total wall time is fsync-dominated.

XXH3 checksums on 8 KB pages run at multi-GB/s on modern CPUs — roughly 1–3 µs per
page. Rarely the bottleneck, but worth knowing the floor.

---

## Decision Table

When the user reports a Chisel performance issue, walk this table.

| Symptom | First check | Lever |
|---------|-------------|-------|
| Throughput poor on bulk insert | Commits per N inserts | [Lever 1: Transaction Batching](#lever-1-transaction-batching) |
| Throughput poor on bulk delete | Per-handle vs `delete_many` | [Lever 1](#lever-1-transaction-batching) (and use `delete_many`) |
| Latency p50 fine, p99 bad | Cache hit rate, defrag pending | [Lever 2: Cache Tuning](#lever-2-cache-tuning), schedule defrag |
| `CacheFull` errors | Transaction working set vs cache_size | [Lever 2](#lever-2-cache-tuning) — increase cache, or split transaction |
| File growing without bound | Defrag never running, or savepoint blocking reuse | [Lever 3: Defrag Scheduling](#lever-3-defrag-scheduling) |
| Reads slow, writes fine | Cold cache, HT depth, overflow chain length | [Lever 4: Read-Path Tuning](#lever-4-read-path-tuning) |
| Tests slow | Filesystem mode for in-memory work | [Lever 5: In-Memory for Tests](#lever-5-in-memory-for-tests) |
| Python-bound workload slow | GIL release, value marshaling | [Lever 6: PyO3 Binding](#lever-6-pyo3-binding) — see `references/python-binding.md` |
| Profile shows wide `__rust_alloc` | Per-op `Vec<u8>` for read results | [Lever 7: Allocation Reduction](#lever-7-allocation-reduction) |

---

## Levers (priority order)

These are listed by expected impact for typical workloads, biggest first.

### Lever 1: Transaction Batching

**The single highest-impact lever.** Two-to-three fsyncs per commit means N operations
in 1 commit cost ~2 fsyncs total; N operations in N commits cost 2N fsyncs. For N=1000,
that's the difference between ~1 ms and ~2 seconds at NVMe latencies, much worse on
rotational storage.

```rust
// BAD — fsync × 2000
for batch in inputs.chunks(1) {
    db.begin()?;
    for v in batch { db.allocate(v)?; }
    db.commit()?;
}

// GOOD — fsync × 2
db.begin()?;
for v in inputs { db.allocate(v)?; }
db.commit()?;
```

The constraint: a single transaction's working set must fit within `cache_size × 8`
(the hard ceiling). For very large bulk operations, choose batch sizes that fill the
cache without tripping `CacheFull`. See `references/transaction-batching.md` for sizing
guidance.

`delete_many(handles)` exists for the same reason — it batches the handle-table COW
work into one transaction-internal pass instead of N.

### Lever 2: Cache Tuning

`cache_size` (in pages, default 1024 = 8 MB) is the second-biggest knob. Two distinct
working sets matter:

- **Read working set** — the handle-table interior pages plus frequently-read data pages
  the application touches. If this exceeds `cache_size`, the LRU thrashes and reads
  generate I/O.
- **Transaction working set** — pages dirtied during one commit's worth of work. Each
  dirty page pins itself in the cache (cannot be evicted while dirty), so a long
  transaction can drive cache occupancy past `cache_size` toward the `cache_size × 8`
  hard ceiling.

```rust
let options = Options {
    cache_size: 16384,    // 128 MB at 8 KB pages
    ..Options::default()
};
```

Sizing rules of thumb:

- For point-lookup workloads (small reads, few transactions): cache the active handle
  table — at depth 2 with full coverage, ~510 leaf pages × 8 KB ≈ 4 MB.
- For mixed read/write with bulk transactions: `cache_size` should comfortably hold
  one transaction's worth of dirty pages. Profile with `Stats` to size.
- For pure write workloads: cache only matters for the handle-table-COW reads;
  `cache_size = 1024` is usually plenty.

Full guide: `references/cache-tuning.md`.

### Lever 3: Defrag Scheduling

Slot tombstones from `delete()` accumulate on data pages until `defrag()` reclaims
them. Running defrag too often is write amplification (pages get rewritten); never
running it bloats the file and slows reads (more pages, worse locality).

**Heuristic**: run defrag when `stats.handle_count` divided by
`(stats.total_pages - superblocks - HT pages - 1)` falls below ~50% of the steady-state
density observed early in the file's life, OR when the file size exceeds a calibration
threshold.

```rust
db.begin()?;
let stats = db.defrag(DefragOptions {
    sparse_threshold: 0.25,  // pack pages with <25% live slots
    max_pages: 1000,         // bound the work per pass
})?;
db.commit()?;
```

Schedule it during low-activity windows; never on the hot path.

### Lever 4: Read-Path Tuning

Reads cost 1–3 page reads through the handle table plus 1–N for the value. The handle
table is a fixed-fanout radix tree (510 leaves × 1021^d), so depth grows logarithmically
with the live-handle count. At ~520k handles, depth = 2 — three page reads per lookup.
At ~531M, depth = 3 — four page reads per lookup.

In practice, depth-1 and depth-2 trees fit easily in the LRU. Read latency below the
fsync floor only matters for hot lookups; for those, the cache settles within a small
number of pages and reads become memory-bound.

Optimization avenues, in order of leverage:

1. **Cache hit rate.** See [Lever 2](#lever-2-cache-tuning).
2. **Avoid re-reading the same value.** Callers can cache hot reads outside Chisel.
3. **Inline values when feasible.** Values under `MAX_INLINE_VALUE` (~`PAGE_BODY_SIZE`)
   stay in the data-page slot — one extra page read at most. Larger values incur an
   overflow chain walk. If a workload has many values just over the inline threshold,
   consider whether they can be split or compressed.

### Lever 5: In-Memory for Tests

`Chisel::open_in_memory()` runs the same engine against `Vec<u8>`-backed `PageIo`. **No
filesystem, no fsync.** Every test that doesn't specifically exercise crash recovery or
flock should use it.

```rust
let mut db = Chisel::open_in_memory()?;   // not a TempDir; no I/O
```

A test suite running against the file backend pays the full fsync cost on every commit;
the same suite against in-memory mode is bound by CPU only and runs an order of
magnitude faster. The code paths are identical except `PageIo`, so coverage is
substantially preserved.

For benchmarking: in-memory mode isolates Chisel's CPU/memory cost from the storage
layer. If you want to measure pure engine overhead (handle-table walk, slot packing,
checksum cost), use in-memory. If you want to measure end-to-end committed throughput
including fsync, use the file backend on the storage you care about.

### Lever 6: PyO3 Binding

The Python binding adds:

- One PyO3 call boundary per operation.
- One `Vec<u8>` ↔ `PyBytes` conversion per read (allocates a new `bytes` object).
- The GIL — held by default during the Rust call, which serializes all Python threads.

Optimizations:

- **Release the GIL** for long operations (`commit`, `defrag`, large `read`/`allocate`)
  via `py.allow_threads(...)`. Already done in chisel-py for these paths; verify when
  adding new methods.
- **Bulk methods** — `allocate_many`, `read_many`, `delete_many` exposed at the binding
  layer reduce Python ↔ Rust crossings. The Rust core's `delete_many` is the prototype;
  consider similar batching for high-throughput Python callers.
- **Avoid unnecessary copies** — `PyBytes` already owns its data; for read paths,
  emitting a `PyByteArray` (mutable) or a `memoryview` over an Arc-shared buffer can
  skip a copy when the caller will not retain the bytes long-term. Trade-off: lifetime
  management gets harder.

Full guide: `references/python-binding.md`.

### Lever 7: Allocation Reduction

Chisel allocates more than a typical Rust library because shadow paging is alloc-heavy
by construction. Most allocations are necessary; some aren't.

Audit targets:

- **Per-read `Vec<u8>` for the value.** This is the API contract; most callers want
  ownership. For zero-copy reads, a separate API returning `&[u8]` borrowed from the
  cache could be added — with the constraint that the borrow must end before the next
  mutating call (compiler-enforced via `&self` vs `&mut self`).
- **`Vec<u8>` for transient slot directories.** `data_page.rs` operations should
  manipulate slots directly on the cached page bytes, never copying out and writing
  back. (Verify in code; this is the existing pattern.)
- **Format strings in error paths.** `ChiselError` variants should carry data, not
  formatted strings. Errors are constructed in fatal paths where one extra alloc is
  fine, but verify nothing in the hot path constructs error messages eagerly.
- **Global allocator.** `mimalloc` is a 1-line change with measurable wins on
  alloc-heavy workloads. See `rust-performance/references/allocations.md`.

---

## Don't-Break List

These are architectural commitments. A change that violates any of them is wrong even
if it benchmarks faster. Always verify a proposed optimization against this list before
implementing.

1. **The two-fsync ordering.** Data fsync must complete before superblock fsync starts.
   No exceptions, no batching across that barrier. (See `ARCHITECTURE.md` commit protocol.)
2. **The pre-drain flush.** The extra fsync before `persist_freemap` is there to keep
   `CacheFull` from being raised mid-commit and silently demoted to fatal. Don't
   "optimize it away" without addressing the underlying cache-ceiling interaction. (See I28.)
3. **The poison model.** Any fatal error from inside the commit protocol — including a
   failed fsync — must poison the manager. No retry, no recovery, no silent demotion to
   operational error. (See I1, fsyncgate.)
4. **On-disk format stability within a major.** No byte on disk changes meaning within
   a major version. Page-level evolution goes through the `page_format_version` byte
   (I31); file-level evolution requires a major bump. Reserved bytes are not free real
   estate — they're the forward-compatibility budget.
5. **`PageType = 0x00` reservation.** Zero must remain a non-page-type. A zeroed page
   cannot masquerade as valid; child-pointer zero means "no child" because page 0 is
   always a superblock.
6. **Strict layer dependency.** No upward references, no sideways references. Layer N
   depends only on layers < N. If your optimization needs the dispatcher to know about
   data pages, it's wrong.
7. **Single-writer `&mut self` contract.** No internal `Mutex`, no interior mutability
   for mutating paths, no concurrent transactions. The flock is exclusive even for
   readers — that's deliberate and required by shadow-paging invariants.
8. **Checksum coverage.** Every page on disk must have a verified XXH3 checksum at
   `8184..8192`. No "skip checksum for hot pages" optimization. Checksum is verified on
   load (cache miss) only — that's the right place; don't move it.
9. **Handle stability.** Handles are never reused within a database's lifetime, even
   after delete. The tombstone in the leaf is what guarantees this. Don't reclaim
   tombstoned slots.
10. **The freemap-allocates-before-merging order.** `persist_freemap` allocates the new
    freemap page id BEFORE merging the freed list, because the freed pages are still
    referenced by the on-disk superblock until commit completes. Reordering creates a
    write-overlap window. (See I18.)

---

## Performance Review Checklist

When reviewing a Chisel PR, walk this list explicitly. The general rust-performance
antipattern catalog applies first; this is the Chisel-specific layer.

### Did the change…

1. Add allocations to a hot path — particularly the cache-hit read path?
2. Add a syscall to anything other than `page_io.rs`?
3. Reorder anything in the commit protocol?
4. Touch the on-disk format (any byte in a `[u8; 8192]` page buffer)?
5. Bump or rely on `page_format_version` without an opt-in upgrade path?
6. Introduce `unsafe` at any layer ≤ transaction.rs?
7. Allow a fatal error to propagate without poisoning the manager?
8. Add a code path where checksum validation can be skipped on load?
9. Add interior mutability or internal locking?
10. Introduce a sideways or upward dependency in the layer graph?
11. Change the meaning of an existing operational error (especially `CacheFull`)?
12. Skip the pre-drain flush in commit?
13. Reuse a handle ID after delete?
14. Modify the freemap merge order in `persist_freemap`?
15. Add a code path where dirty pages can escape the cache without flush?

If any answer is yes, the change needs explicit justification — not a benchmark, an
architectural argument. The right home for such an argument is `ISSUES.md`.

### Bench expectations for performance PRs

Every performance PR should include:

- **Criterion or divan benchmark** in the relevant layer's bench file, with before/after
  numbers.
- **Both backends measured** — in-memory (CPU/memory cost) and file (full cost). The
  ratio between them should not change substantially; if it does, the change has shifted
  cost onto the storage layer in some unexpected way.
- **A regression test** in the relevant module's test file that exercises the
  optimization's correctness boundary (e.g., if the change adds a fast path for cache
  hits, a test that confirms the slow path still works).

See `references/benchmarking.md` for the project-specific bench harness layout.

---

## Reference Files

- `references/cost-model.md` — Detailed cost analysis: per-operation, per-layer, and how
  costs compose. Includes a "back-of-envelope" calculator for predicted throughput.
- `references/transaction-batching.md` — Sizing transactions, the `cache_size × 8`
  constraint, when to split a logical batch into multiple commits, and the
  fsync-amortization math.
- `references/cache-tuning.md` — Tuning `cache_size` for read-heavy, write-heavy, and
  mixed workloads. How to read `Stats`, instrument hit rate, and diagnose `CacheFull`.
- `references/benchmarking.md` — Chisel-specific benchmark harness: per-layer bench
  files, in-memory vs file fixtures, criterion patterns for the existing code, and
  CI-stable instruction-count benches via iai.
- `references/python-binding.md` — PyO3 overhead, GIL release patterns, value
  marshaling cost, when to add bulk methods at the binding boundary.
- `references/review-checklist.md` — The full review checklist with rationale for each
  item, suitable for use as a PR-review template.
