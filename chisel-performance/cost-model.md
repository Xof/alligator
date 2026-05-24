# Cost Model Reference

Detailed analysis of where time goes in Chisel operations. Use this to predict
throughput before optimizing, and to verify a profiled bottleneck matches the
expected cost shape.

## Table of contents

1. The fsync floor
2. Per-page costs (allocate, write, checksum)
3. The handle-table walk
4. Slot packing and the data page
5. Overflow chains
6. The commit protocol cost
7. The rollback cost
8. Defrag cost
9. Back-of-envelope throughput calculator
10. When the model doesn't match — debugging cost surprises

---

## 1. The fsync floor

Every commit performs at least 2 fsyncs (data, then superblock) and almost always 3
(the pre-drain flush before `persist_freemap`). The fsync cost depends on the storage:

| Storage | Typical fsync cost |
|---------|-------------------|
| NVMe SSD (consumer, well-cached) | 100–500 µs |
| NVMe SSD (saturated or no cache) | 1–5 ms |
| SATA SSD | 1–10 ms |
| Rotational HDD | 5–20 ms |
| Network filesystem | 10–500 ms (highly variable, often slower) |
| In-memory mode | 0 (no syscall) |

These numbers dominate any committing workload. A workload that does N independent
commits on a typical NVMe is bounded at roughly `N / 0.001 s = 1000 commits/s` even if
the actual work per commit is microseconds. The same workload batched into a single
commit runs at CPU speed.

The implication: **transactions are the unit of throughput, not operations.** A workload
specification must include "how many ops per commit" to be meaningful.

### Macos `F_FULLFSYNC` vs `fsync(2)`

On macOS, `fsync(2)` does *not* flush the drive's write cache to non-volatile storage —
it only flushes the kernel's buffer to the drive. `F_FULLFSYNC` is the real durability
primitive but is several times slower (often 5–20 ms even on fast SSDs). Chisel uses
`fsync(2)` and accepts the trade-off; this matches PostgreSQL's default
(`wal_sync_method = fsync`). For workloads that need true power-loss durability on
macOS, the application must opt in via a different sync method — currently not exposed.

---

## 2. Per-page costs

For every page-level operation, the costs that compose:

| Operation | CPU | I/O | Allocation |
|-----------|-----|-----|------------|
| `PageCache::get` (hit) | LRU touch (~30 ns) | 0 | 0 |
| `PageCache::get` (miss) | XXH3 verify (~1 µs) + LRU insert | 1 page read (50 µs–1 ms) | 1 page buffer (8 KB) |
| `PageCache::new_page` | LRU insert | 0 (lazy write) | 1 page buffer |
| `PageCache::flush` (per dirty page) | XXH3 compute (~1 µs) | 1 page write (50 µs–1 ms) | 0 |
| `XXH3::hash` of 8 KB | ~1 µs at 8 GB/s | 0 | 0 |

### XXH3 cost in context

XXH3 on modern x86 (with SSE2 baseline) runs at multi-GB/s; on Apple Silicon with NEON,
similar. For 8 KB pages, that's roughly 1 µs per page. In a 1000-page transaction,
checksum cost is ~1 ms — meaningful, but well below fsync cost.

For workloads where checksum *is* the bottleneck (extremely fast storage, very large
in-memory transactions), the constant is `target-cpu` dependent. A native build with
AVX2 or AVX-512 can be substantially faster than the baseline. SIMD-optimized XXH3
variants exist (`twox-hash` with the `xxh3` feature uses them) but the gain over
scalar XXH3 on an 8 KB buffer is modest because the hash is already vectorized.

---

## 3. The handle-table walk

The handle table is a fixed-fanout radix tree:

- Leaf capacity: 510 entries × 16 bytes
- Interior fanout: 1021 children × 8 bytes

Capacity at depth `d`: `510 × 1021^d`.

| Live handles | Tree depth | Page reads per lookup |
|--------------|-----------|----------------------|
| ≤ 510 | 0 (root is leaf) | 1 |
| ≤ ~520 K | 1 | 2 |
| ≤ ~531 M | 2 | 3 |
| ≤ ~542 G | 3 | 4 |

For all realistic workloads, depth ≤ 2. Depth-3 trees are theoretical; getting there
takes hundreds of millions of live handles.

### Cost composition per lookup

```
read(h) = HT walk + data-page access + value extraction

HT walk (cold cache):
  d page reads, each:
    - 1 page I/O (50 µs–1 ms)
    - 1 XXH3 verify (1 µs)
    - LRU insert (30 ns)
  → ~50 µs–3 ms total for d=1..3

HT walk (warm cache):
  d LRU lookups (~30 ns each)
  → ~100 ns total

Data page access:
  - 1 page read (cold) or LRU lookup (warm)
  - slot directory walk: O(1) (slot index is direct)
  - 1 Vec<u8> allocation for return

Total cold: 100 µs–4 ms (storage-bound)
Total warm: ~1 µs (allocation-bound)
```

The point: **once the working set is cached, reads become allocation-bound.** The
remaining knob is the per-read `Vec<u8>` allocation. See lever 7 in the main skill.

---

## 4. Slot packing and the data page

A data page (`PageType::Data`, 8 KB):

```
+--- 16 bytes header
|
|--- slot directory (grows forward)
|    each entry: ~6-8 bytes (offset + length + flags)
|
|--- ... gap ...
|
|--- packed value data (grows backward)
|
+--- 8 bytes XXH3 checksum
```

Capacity: `PAGE_BODY_SIZE` ≈ 8160 bytes split between directory and packed values.
A page might hold:

- ~1000 small (4-byte) values + their directory entries
- ~10 medium (700-byte) values
- 1 value approaching the inline threshold

Slot packing means writes amortize page allocation: the first `allocate()` to a fresh
data page pays the new-page cost; subsequent allocates that fit reuse the same page
(via `claim_page`) and only dirty it once per commit.

### Implication for write batching

If you're writing 1000 small values:

- 1000 `allocate()` calls within one transaction → 1 (or few) data pages dirtied,
  ~1000 slot directory entries, 1 fsync amortized over all of them.
- 1000 `allocate()` calls across 1000 transactions → each commit allocates a new data
  page (the previous one is committed and not extended), 1000 data pages total, 2000
  fsyncs.

Same data, two orders of magnitude difference.

### When slots tombstone

`delete()` writes a tombstone in the handle-table leaf, but the data-page slot stays
allocated. Tombstone slots accumulate until `compact()` (called by `defrag()`)
rewrites the page. A workload of insert-then-delete that never runs defrag will see
data pages never go fully empty, so the freemap won't reclaim them — an accumulation
pattern visible as "file grows even though handle count stays low."

---

## 5. Overflow chains

Values larger than `MAX_INLINE_VALUE` (~`PAGE_BODY_SIZE`, just under 8 KB) get split
into a singly-linked overflow chain. Each overflow page holds `OVERFLOW_PAYLOAD` =
8152 bytes of value data plus a `next_page` pointer.

| Value size | Chain length | Page reads (cold) | Pages dirtied (alloc) |
|-----------|--------------|-------------------|----------------------|
| 8 KB | 1 | 1 | 1 |
| 100 KB | 13 | 13 | 13 |
| 1 MB | 129 | 129 | 129 |
| 10 MB | 1287 | 1287 | 1287 |

For very large values, overflow walking is sequential and storage-bound; the cache
helps only if the chain has been read before. For repeated reads of large overflow
values, callers should consider whether they want to keep their own decoded
representation outside Chisel.

### Cycle detection

The walker bounds the chain by `total_length / OVERFLOW_PAYLOAD` (I14). A corrupt
chain that loops forever cannot exceed this bound; the walker returns `CorruptPage`
rather than spinning. Cost: a counter increment per page, negligible.

---

## 6. The commit protocol cost

Total cost for a commit dirtying M pages affecting H handles:

```
Commit cost ≈
   + flush dirty pages    : M × (XXH3 + write) + fsync_data        [pre-drain]
   + persist_freemap      : 1 alloc + 1 XXH3 + 1 write + fsync     [freemap]
   + write superblock     : 1 XXH3 + 1 write + fsync               [superblock]
   + handle-table COW (during prior allocates/updates):
                          : H × tree_depth × (XXH3 + write)        [already counted in M]

Wall time:
   = ~ M × (1 µs + page_write_cost) + 3 × fsync_cost
   ≈ 3 × fsync_cost                  (for moderate M)
```

The "moderate M" caveat: when M is large enough that writing M pages takes longer than
fsync, the page-write cost matters. On NVMe at 1 GB/s effective sequential write, M
must exceed ~1000 pages (8 MB) before page-write time dominates fsync. Most commits
are far below this.

---

## 7. The rollback cost

Rollback is essentially free:

- Pages allocated during the transaction (id ≥ watermark) get truncated from the file
  and dropped from cache.
- Pages reused from the freemap (id < watermark) get their dirty cache entries
  invalidated; the on-disk page still has the previously-committed content.
- No fsync (rollback isn't durable — the previously-committed state is what's durable).

The dominant cost is cache invalidation for the dirty entries: O(dirty_pages) × LRU
removal. Microseconds for typical transactions.

This is why "always begin/commit even for one operation" is fine for code clarity:
rollback is the cheap path. The expensive thing is committing.

---

## 8. Defrag cost

`defrag()` examines pages whose live-slot count is below `sparse_threshold`, then
re-inserts their live values into other pages. For each value relocated:

- 1 read from old page (LRU lookup, usually warm during defrag)
- 1 write to new (or existing not-full) data page (dirties one page)
- 1 handle-table COW path to update the entry (depth × (XXH3 + write))

After all values are relocated, source pages become fully empty and feed the freemap
on commit.

Cost = O(values_relocated) × (HT depth + 2) page operations + final commit cost.

`max_pages` (poorly named — it's `max_values` in practice; see C4) bounds the work per
pass, which lets defrag run inside a budget.

---

## 9. Back-of-envelope throughput calculator

For a workload of `R` reads and `W` writes per second, with batch size `B`
(operations per commit) and storage with fsync cost `F`:

```
Required commits/s = (R × 0 + W) / B               (reads don't commit)
Time per commit ≈ 3F + B × per_op_CPU_cost
Required wall time / s = (W / B) × (3F + B × per_op_CPU_cost)
                      = 3FW/B + W × per_op_CPU_cost
```

Setting required wall time ≤ 1 s for sustainability:

```
3FW/B + W × per_op_CPU_cost ≤ 1
```

For NVMe (F ≈ 200 µs), W = 10000 writes/s, per_op_CPU_cost ≈ 5 µs:

```
3 × 200 µs × 10000 / B + 10000 × 5 µs ≤ 1 s
6 s / B + 0.05 s ≤ 1 s
6 / B ≤ 0.95
B ≥ 6.3
```

So at B ≥ 7 ops/commit, this workload is sustainable on NVMe. At B = 1, it needs
about 6× the CPU/storage budget — i.e., it won't keep up.

For rotational (F ≈ 10 ms):

```
3 × 10 ms × 10000 / B + 0.05 s ≤ 1 s
300 s / B ≤ 0.95
B ≥ 316
```

Same workload requires batches of 316+. This is the difference durability infrastructure
makes; it's not a bug, it's the cost of two-fsync shadow paging.

---

## 10. When the model doesn't match

If profiling shows costs that don't match the model:

| Symptom | Likely cause |
|---------|-------------|
| Commit cost much higher than 3 × fsync | Excessive M (too many dirty pages); cache thrashing; or `CacheFull` → poison |
| Read cost much higher than expected | Cold cache; HT depth higher than expected (look at `Stats`) |
| Allocation count higher than expected | Per-read `Vec<u8>` × N reads; format strings in errors; missed `with_capacity` |
| File growth without corresponding handle growth | Defrag never ran; or a long savepoint stack disabled freemap reuse |
| Fsync cost orders of magnitude off | Wrong storage assumption; check `iostat` or equivalent for actual write latency |
| In-memory benchmark slower than expected | `cache_size` too small for working set; check `CacheFull` |

When the model doesn't match, start with `dhat` (allocations) or `samply` (CPU time)
on the in-memory backend to isolate engine cost from storage cost. See
`benchmarking.md` for setup.
