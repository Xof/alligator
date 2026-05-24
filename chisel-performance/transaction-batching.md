# Transaction Batching Reference

The single highest-leverage optimization for any committing Chisel workload. This file
goes deep on how to size batches, when to split them, and the failure modes.

## Table of contents

1. The fsync amortization argument
2. The `cache_size × 8` ceiling
3. Sizing batches: by count, by size, by working-set
4. When to split a logical batch
5. The savepoint trick (and its limits)
6. Pathological cases: huge values, many overflow chains
7. Bulk methods at the API boundary
8. Examples

---

## 1. The fsync amortization argument

Restated from the main skill, with the math made explicit.

A commit costs ~3 × fsync_cost (data + freemap + superblock; pre-drain adds the first
one). For storage with fsync cost F, N commits with B operations per commit cost:

- **Wall time on commits**: `N × 3F`
- **Wall time on operations**: `N × B × per_op_CPU_cost`

If we hold `N × B = total_ops` constant and vary B:

| Batch size B | Number of commits | Time on fsync | Time on CPU | Total |
|--------------|------------------|--------------|-------------|-------|
| 1 | total_ops | total_ops × 3F | total_ops × c | dominated by fsync |
| 100 | total_ops / 100 | total_ops × 3F / 100 | total_ops × c | F-cost shrinks 100× |
| 10000 | total_ops / 10000 | total_ops × 3F / 10000 | total_ops × c | F-cost negligible |

The CPU work is the same regardless of batching. The fsync work scales with 1/B until
B grows large enough that the next constraint binds.

---

## 2. The `cache_size × 8` ceiling

The constraint that keeps you from making B arbitrarily large: every page dirtied in
the transaction stays pinned in the cache (cannot evict while dirty). The cache has a
hard ceiling at `cache_size × 8` (default 1024 × 8 = 8192 pages = 64 MB) beyond which
allocation returns `CacheFull`.

So B is bounded by the dirty-page count fitting under `cache_size × 8`.

### How many dirty pages does a transaction generate?

For W operations dirtying handles uniformly:

- W new values inline-packed into K data pages (K depends on value size; for small
  values, K << W)
- HT COW pages: roughly W × tree_depth at worst, often much less because adjacent
  handle ids share leaf and interior pages
- 1 freemap COW (negligible)

Worst-case: each operation touches a unique HT path. That's roughly W × tree_depth
dirty pages. At depth 2, W × 3 ≈ 3W dirty pages.

Best-case: operations cluster, sharing HT leaf and interior pages. That's roughly
W / 510 leaves + 1 root ≈ W/500 dirty pages.

In practice, a sequential allocate workload (handle ids assigned monotonically) is
near best-case — consecutive handles share the same leaf, and dirtying the leaf once
affects 510 entries.

### Picking B

For default `cache_size = 1024`:

- Hard ceiling: 8192 dirty pages.
- Best-case (sequential allocate, small values): B can be in the tens of thousands per
  commit before approaching ceiling.
- Worst-case (random handle distribution, many distinct HT leaves): B should stay
  below ~2000.

If your workload pattern is unknown, **pick B such that `B / 100 × tree_depth + B /
slot_pack_factor` stays well under `cache_size × 8`**, then verify with `Stats` after
a representative run.

For sustained throughput targets, a common pattern: `cache_size = 16384` (128 MB),
`B = 50000`. This handles tens of thousands of operations per commit comfortably and
fsync cost amortizes to negligible.

---

## 3. Sizing batches

Three approaches:

### By operation count

Pick B as a constant. Simple, works fine for uniform-sized values.

```rust
const BATCH_SIZE: usize = 10_000;

for chunk in inputs.chunks(BATCH_SIZE) {
    db.begin()?;
    for v in chunk { db.allocate(v)?; }
    db.commit()?;
}
```

### By accumulated value size

Better when value sizes vary widely (some inline, some causing many overflow pages).

```rust
const BATCH_BYTES_TARGET: usize = 32 * 1024 * 1024;   // 32 MB

let mut accumulated = 0;
db.begin()?;
for v in inputs {
    db.allocate(v)?;
    accumulated += v.len();
    if accumulated >= BATCH_BYTES_TARGET {
        db.commit()?;
        db.begin()?;
        accumulated = 0;
    }
}
db.commit()?;
```

### By handle-count delta

Useful when reads and writes mix and you want to bound HT working set.

```rust
const HANDLES_PER_COMMIT: usize = 5000;

let starting_handles = db.stats()?.handle_count;
db.begin()?;
loop {
    let n = db.stats()?.handle_count - starting_handles;
    if n >= HANDLES_PER_COMMIT { break; }
    db.allocate(next_value()?)?;
}
db.commit()?;
```

---

## 4. When to split a logical batch

Sometimes a single logical "transaction" is too large to fit. Examples:

- Loading a 10 GB file into Chisel as overflow chains: ~1.3M pages dirty, won't fit
  any sane cache.
- Bulk-rewriting a large dataset for a schema change.

Splitting strategy depends on the consistency requirements:

### If atomicity isn't required

Split into independent commits. Each commit is a separate atomic unit; if the process
crashes mid-load, partial work is durable and you resume from where you left off.

```rust
for chunk in huge_input.chunks(BATCH_SIZE) {
    db.begin()?;
    for v in chunk { db.allocate(v)?; }
    db.commit()?;   // each commit independently durable
}
```

Use a separate "progress" handle (or named root) to track which chunks have been
applied. On restart, read it, resume.

### If atomicity IS required

You can't split. The whole work must be one transaction. If it doesn't fit:

- **Increase `cache_size`.** The hard ceiling scales 8× cache_size; if you need 1M
  dirty pages, set `cache_size = 131072` (1 GB).
- **Decrease the per-operation page footprint.** Are you allocating overflow chains
  unnecessarily? Can values be split?
- **Reconsider the requirement.** Is true atomicity needed, or just "either fully
  applied or detectable as half-applied"? The latter can use a marker handle.

There is no "split atomic transaction across commits" feature — that would require
WAL semantics, which Chisel deliberately doesn't have.

### Savepoints don't help here

Savepoints are inside one transaction. They don't reduce the dirty-page working set
because all savepoint scopes' dirty pages remain pinned until the outer commit (or
rollback). They're useful for **logical** partial rollback, not for **memory** management.

---

## 5. The savepoint trick (and its limits)

Savepoints are useful when you want most of a transaction to commit, but specific
operations to be rollbackable on a soft failure:

```rust
db.begin()?;
for batch in batches {
    db.savepoint("batch_attempt")?;
    match try_apply_batch(&mut db, batch) {
        Ok(()) => db.release("batch_attempt")?,
        Err(e) if e.is_recoverable() => {
            db.rollback_to("batch_attempt")?;
            log_skip(batch, e);
        }
        Err(e) => return Err(e),
    }
}
db.commit()?;
```

This commits all successful batches as a single durable unit while skipping the
failed ones. Use case: ETL pipelines where some records are malformed.

### What savepoints don't do

- They don't reduce the dirty-page working set (still pinned until outer commit).
- They temporarily disable freemap reuse — pages freed inside a savepoint scope cannot
  be reallocated until the savepoint resolves. Long savepoint stacks plus heavy
  delete-and-allocate causes file growth.
- They have no fsync semantics. Only `commit()` is durable.

---

## 6. Pathological cases

### Single huge value

A 1 GB value generates 131,000 overflow pages, all dirtied in one transaction. With
default `cache_size = 1024`, this trips `CacheFull` long before completion.

Solutions:

- **Stream as chunks.** Decompose the value at the application layer into many
  smaller values with a manifest handle pointing at them. Reads recompose.
- **Increase cache.** `cache_size = 16384` gives a 128 MB hard ceiling — still not
  enough for 1 GB of overflow pages.
- **Don't.** Chisel's data model is "many handles, mostly small." Multi-GB values
  are at the edge of the design envelope.

### Many overflow chains in one transaction

100 values of 100 MB each: 1.3M overflow pages. Same problem as above; same solutions.

### Wide handle-id distribution in one transaction

Touching 1M handles spread across the full radix tree dirties O(1M) HT pages even if
each operation is small. The HT structure isn't designed for "rewrite half the leaves
in one transaction." Defrag is the only operation that legitimately does this kind of
sweep, and it's bounded by `max_pages`.

If a workload genuinely needs to rewrite many handles, batch them in commit-sized
chunks ordered by handle id. Adjacent handle ids share leaves, so sorted iteration
keeps the dirty-page count down.

---

## 7. Bulk methods at the API boundary

`delete_many(handles)` already exists in the Rust core. The pattern: batch the
HT-COW work into one in-transaction pass instead of N separate `delete()` calls.
The savings are mostly CPU (fewer LRU lookups, fewer slot-directory walks), not
fsync — both forms still commit once.

For high-throughput callers (especially the Python binding), consider similar bulk
methods:

- `allocate_many(values: &[&[u8]]) -> Vec<u64>` — amortizes the allocate-data-page
  loop, the HT-leaf walk, and the FFI boundary cost (huge for chisel-py).
- `read_many(handles: &[u64]) -> Vec<Vec<u8>>` — sorts handles to maximize HT-leaf
  cache locality.
- `update_many(pairs: &[(u64, &[u8])]) -> Result<()>` — same.

These are at the **API boundary**, not the storage layer. They don't change the
underlying engine's per-op cost; they change the per-call overhead. For Rust callers,
the overhead saved is small (a few function calls). For Python callers, it's the
PyO3 crossing per call — see `python-binding.md`.

---

## 8. Examples

### Bulk insert from a file

```rust
use std::io::BufRead;

const BATCH: usize = 10_000;

let mut db = Chisel::open("data.db".as_ref(), Options::default())?;
let mut count = 0;

db.begin()?;
for line in input.lines() {
    let line = line?;
    db.allocate(line.as_bytes())?;
    count += 1;
    if count % BATCH == 0 {
        db.commit()?;
        db.begin()?;
    }
}
db.commit()?;
```

For 10M lines on NVMe, this runs at roughly `(10M / 10000) × 200 µs = 200 ms` of
fsync cost plus CPU. The same code with `BATCH = 1` runs at `10M × 200 µs = 2000
seconds`. Five orders of magnitude.

### Read-modify-write loop

```rust
db.begin()?;
for h in handles_to_update {
    let old = db.read(*h)?;
    let new = transform(&old);
    db.update(*h, &new)?;
}
db.commit()?;
```

Reads happen inside the transaction (legal; reads work in or out of one). The cache
warms from the reads; updates dirty new pages. One commit; one fsync amortization.

If `handles_to_update` is huge, the same `BATCH`-then-commit pattern applies. Sort by
handle id first to maximize HT-leaf cache locality.

### When a batch is too large

Symptom: `Err(ChiselError::CacheFull)` from `allocate()` or `update()`.

```rust
match db.allocate(&value) {
    Ok(h) => h,
    Err(e) if matches!(e, ChiselError::CacheFull) => {
        db.commit()?;       // commit what we have
        db.begin()?;        // start fresh
        db.allocate(&value)?    // retry (the cache is now drained)
    }
    Err(e) => return Err(e),
}
```

Or, better, size B more conservatively in advance.
