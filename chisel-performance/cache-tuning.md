# Cache Tuning Reference

The Chisel page cache is an LRU over fixed 8 KB pages with a soft limit (`cache_size`)
and a hard ceiling (`cache_size × 8`). This file explains how to size it, how to
diagnose problems, and what each failure mode looks like.

## Table of contents

1. The cache geometry
2. Soft limit vs hard ceiling — what each means
3. Sizing for read-heavy workloads
4. Sizing for write-heavy workloads
5. Sizing for mixed workloads
6. Diagnosing `CacheFull`
7. Diagnosing thrashing
8. Why hits skip checksum validation
9. The `flock` interaction

---

## 1. The cache geometry

```
PageCache
├── soft limit: cache_size pages         (default 1024 = 8 MB)
├── hard ceiling: cache_size × 8 pages   (default 8192 = 64 MB)
└── eviction policy: LRU, dirty pages pinned
```

The soft limit is what eviction targets when reading new pages: when the cache reaches
`cache_size`, `maybe_evict` walks the LRU and evicts clean pages.

The hard ceiling is what `new_page` and `claim_page` enforce when allocation cannot
find room — even if all pages are dirty (and thus uneveable), allocation refuses to
exceed the ceiling and returns `ChiselError::CacheFull`.

The 8× factor is the "headroom for one big transaction" — the soft limit governs
read working set, the ceiling governs transaction working set, and the ratio
balances "modest steady-state memory" with "transactions that grow dynamically don't
trip on small numbers."

---

## 2. Soft limit vs hard ceiling

### Hitting the soft limit: clean pages get evicted

Normal operation. A read pulls a page in, an older page gets dropped. The dropped
page's content is gone from memory but durable on disk; re-reading it pays one disk
read.

A workload that sustainably operates above the soft limit is fine — it just means
some reads incur I/O. The question is whether the hot working set fits.

### Approaching the hard ceiling: dirty pages can't evict

If a transaction dirties so many pages that clean pages run out, the cache hits the
hard ceiling. Allocation returns `CacheFull`. The transaction must commit or roll
back to release the dirty pin.

`CacheFull` is operational, not fatal. The `Chisel` handle isn't poisoned. But
encountering it mid-transaction means the work-so-far isn't committable as the same
transaction; the caller has to decide what to do.

### Why is the ratio 8 and not, say, 2 or 100?

8 was chosen empirically:
- Small enough that a runaway operation fails fast rather than consuming all memory.
- Large enough that normal transactions with reasonable batching (a few thousand
  operations, multi-thousand dirty pages) don't routinely hit it.
- A power of 2 for clean reasoning about cache size.

It's configurable via `cache_size`; a workload that needs more headroom should
increase the soft limit, not change the ratio.

---

## 3. Sizing for read-heavy workloads

For a workload that reads many handles and rarely writes, the cache works as a
classic page cache for the on-disk B-tree-like structure.

Hot working set components:

- **Handle-table interior pages.** Always cached if accessed. At depth 2, the root
  is one page; at depth 3, root + ~10 frequently-accessed level-1 interiors. All in,
  ≤ a few dozen pages.
- **Handle-table leaf pages.** One per ~510 handles. For a workload reading from N
  handles uniformly, working set is ~N/510 leaves.
- **Data pages.** One per ~10–1000 values depending on value size. For a workload
  reading from N handles uniformly, working set is at most N data pages, often much
  fewer if multiple values share pages.
- **Overflow pages.** Only relevant if the workload includes large values; each
  large-value read pulls in a chain.

Quick sizing: `cache_size ≥ 2 × hot_set` is a comfortable target. For a workload
with 100,000 hot handles and small values:

- HT leaves: ~200 pages
- HT interiors: ~10 pages
- Data pages (assuming ~500 small values per page): ~200 pages
- Total hot set: ~410 pages
- Recommended `cache_size`: ~1000 pages (default is fine)

For 10M hot handles with similar value sizes:

- HT leaves: ~20,000 pages
- HT interiors: ~30 pages
- Data pages: ~20,000 pages
- Total: ~40,000 pages
- Recommended `cache_size`: ~80,000 pages = 640 MB

If the hot set doesn't fit, reads incur I/O at the ratio "(hot_set - cache_size) /
hot_set" of all reads. Below ~50% capture, the cache is essentially not helping.

---

## 4. Sizing for write-heavy workloads

For write-heavy workloads, the cache's read-side role is minor (HT walks during
allocate are usually warm because allocates touch consecutive handle ids). The
dominant concern is the **transaction working set**.

Per the cost model, a transaction with W write operations dirties roughly:

- Best case: W / 500 + tree_depth pages
- Worst case: W × tree_depth pages (each operation touches a unique HT path)

For W = 10,000 in a sequential allocate workload at depth 2:
- Best: 20 + 2 = 22 dirty pages
- Worst: 30,000 dirty pages

The hard ceiling at default `cache_size = 1024` is 8192 pages. The worst case fails;
the best case is fine. In practice, sequential bulk inserts are best-case.

### Sizing for the worst case

If you can't predict the access pattern, size for worst case: `cache_size ≥ W × tree_depth / 8`.

For W = 10,000 at depth 2: `cache_size ≥ 2500`. Set `cache_size = 4096` for headroom.

### When the cache becomes the bottleneck

If you observe `CacheFull` errors, the options in order of preference:

1. **Reduce W per transaction** (commit more often). Costs one extra fsync each time
   you split.
2. **Increase `cache_size`.** Linear in memory cost, no other downside.
3. **Defragment more aggressively** if HT pressure is from accumulated tombstones.

---

## 5. Sizing for mixed workloads

Mixed = both reads and writes contend for the same cache. The dominant question:
which side is hotter?

- If read working set is larger: size for the read side. The transaction working set
  shares the cache; if the soft limit is comfortably above read hot set, transactions
  have ~7× cache_size of headroom before tripping the ceiling.
- If transaction working set is larger: size for the transaction. The read side gets
  whatever fits after dirty pages settle.

A workload that reads 1M hot handles AND writes 5K-op transactions on a similar
distribution: `cache_size = 4096` gives 32 MB soft limit (covers most of the read
working set) and 256 MB hard ceiling (more than enough for 5K-op transactions even
worst-case).

When in doubt, **measure**. The `Stats` snapshot (`db.stats()?`) gives you
`total_pages` (file size) and `handle_count`; combine with profiling cache hit rate
under representative load.

---

## 6. Diagnosing `CacheFull`

`CacheFull` means: a transaction allocated to the point where every page in the cache
is dirty, the hard ceiling was reached, and the next allocation refused.

Investigation order:

1. **Confirm the transaction was atypically large.** `Stats` before/after the failure
   shows `total_pages` delta, which approximates the dirty-page count.
2. **Check for unexpected page dirtying.** Is the operation that hit `CacheFull`
   doing something that produces more dirty pages than the operation count suggests?
   E.g., reading via a path that triggers HT COW (it shouldn't — reads are read-only).
3. **Check for savepoint stacks.** A savepoint disables freemap reuse during its
   scope; allocating-then-freeing within a savepoint inflates dirty-page count
   because frees don't reclaim immediately.

The fix:

- If the transaction is legitimately large: increase `cache_size`, or split into
  smaller transactions.
- If the transaction is an unexpected size: there's a bug or unanticipated
  amplification. Profile with `dhat` to see what's allocating cache slots.

---

## 7. Diagnosing thrashing

Thrashing = the cache is too small for the working set, so reads constantly evict
each other and re-fetch.

Symptoms:

- Read latencies are bimodal: many fast (cache hit) and many slow (cache miss + I/O).
- Increasing `cache_size` substantially reduces wall-clock time for the same workload.
- Profile shows wide `PageIo::read_page` stacks even on workloads with stable working
  sets.

Confirmation: instrument cache hit rate. There isn't a built-in counter (current
Chisel doesn't expose hit/miss stats from `PageCache`), but you can add one for the
profiling session — wrap `cache.get` to count hits and misses, log the ratio after a
representative run.

Fix: increase `cache_size` until the hit rate plateaus (typical good rate: >95%).

### When to add a built-in cache-stats counter

If diagnosing cache behavior becomes routine, exposing `pages_read`, `pages_evicted`,
and `cache_hit_ratio` from `PageCache` (or via a debug build feature flag) is a small,
non-format-affecting change. The current code doesn't expose these, but adding them
behind a feature flag is well-scoped.

---

## 8. Why hits skip checksum validation

`PageCache::get` validates checksums on cache miss only — once the page is in the
cache, reads don't re-verify. The reasoning:

- The exclusive `flock` ensures no other process modifies the file while Chisel holds
  it open. The on-disk bytes can't change underneath the cache.
- The cache itself is in-process memory; corruption there indicates a far more
  serious problem (memory error, undefined behavior in caller code) that checksumming
  doesn't recover from.
- Re-validating on every read would add ~1 µs per access, multiplying read cost.

Don't change this. If you want defense against in-memory corruption, ECC RAM and
process isolation are the right answer, not application-layer checksums on every
read.

---

## 9. The `flock` interaction

`PageCache` assumes the on-disk file isn't being modified by another process —
`PageIo` enforces this with exclusive `flock`. This is what makes it safe to skip
checksum validation on cache hits.

If you're tempted to relax `flock` to allow shared readers (e.g., for some
high-concurrency optimization): don't. The skip-on-hit checksum behavior is built on
the flock invariant. Removing flock without adding per-read checksumming would let
concurrent writers corrupt the cache without detection.

This is a place where "cheap reads" and "single-writer" are tightly coupled. They're
both load-bearing for shadow paging's recovery guarantees.
