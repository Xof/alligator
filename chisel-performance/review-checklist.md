# Review Checklist (Chisel Performance PRs)

A structured walkthrough for reviewing PRs that touch performance-sensitive paths in
Chisel. Each item has a why, a what-to-look-for, and (where useful) a code pattern.

The checklist is organized by failure mode. The general `rust-performance/references/antipatterns.md`
catalog applies first; this is the Chisel-specific layer.

## How to use this checklist

For each item, the reviewer answers explicitly: applies / does not apply / unclear.
"Unclear" means the PR description should clarify before merge.

This isn't a substitute for reading the diff. It's a backstop against the failure
modes that have bitten Chisel before, captured in `ISSUES.md` from prior reviews.

---

## Section 1: Durability invariants

### 1.1 Commit protocol ordering

**Why**: The order of operations in `TransactionManager::commit` is what makes shadow
paging crash-safe. Reordering changes the failure semantics.

**Look for**: any change to `commit()`, `persist_freemap()`, or the relative order of
`flush()` calls and `write_page(superblock)`.

**Required ordering**:
1. Pre-drain flush (data pages + fsync)
2. `persist_freemap`: allocate new freemap page id BEFORE merging freed pages
3. Flush the freemap page (+ fsync)
4. Build new superblock with `txn_counter += 1`
5. Write to slot `txn_counter % N`
6. fsync (linearization point)
7. Promote in-memory state

A change that touches any of these requires explicit architectural justification, not
a benchmark.

### 1.2 Pre-drain flush in commit

**Why**: Without pre-drain, `persist_freemap`'s `allocate_data_page` call can hit the
cache hard ceiling and raise `CacheFull` mid-commit, which the commit wrapper poisons.
Pre-drain clears dirty pages so eviction can proceed normally. (See I28.)

**Look for**: removal of the pre-commit `flush()` call, or any "optimization" that
elides the extra fsync.

If a PR claims this is unnecessary, ask: how does `allocate_data_page` succeed when
every cached page is dirty?

### 1.3 Fsync semantics

**Why**: Linux fsyncgate (post-2018) means a failed fsync cannot be safely retried.
The poison model captures this.

**Look for**: any error handling around fsync that retries, downgrades to operational,
or continues on. Failed fsync must poison.

**Pattern**:
```rust
match self.io.fsync() {
    Ok(()) => (),
    Err(e) => {
        self.poison(e.clone());
        return Err(ChiselError::IoError(e));
    }
}
```

NOT:
```rust
// BAD
self.io.fsync()
    .or_else(|_| { self.io.fsync() })   // retry — unsafe
    .ok();                                // ignore — even worse
```

### 1.4 Checksum coverage

**Why**: Every page on disk must have a verified XXH3 checksum. Unverified bytes are
not durable; they're "we hope it's fine."

**Look for**: any code that writes a page without computing its checksum, or any code
that loads a page without validating. The current code does both correctly via
`PageCache`; new code paths must follow the same pattern.

### 1.5 Poison consistency

**Why**: Once poisoned, every Chisel call must return `Poisoned` until reopen. Slipping
a call through that doesn't check the poison flag breaks the recovery contract.

**Look for**: new public methods on `Chisel` or `TransactionManager` that don't call
`check_alive()` or equivalent at entry.

### 1.6 Single-writer contract

**Why**: `&mut self` on every mutator is what makes Chisel's lack of internal locking
sound. Interior mutability for mutating paths breaks this — the type system would no
longer prevent two concurrent transactions.

**Look for**: new fields with `RefCell`, `Cell`, `Mutex` that participate in mutation
paths. (Caches, statistics, and other transient state can use them; transaction state
cannot.)

---

## Section 2: On-disk format stability

### 2.1 Format-affecting changes

**Why**: Within a major version, every byte on disk is sacred. Any change that affects
how a file is interpreted is either a minor bump (additive, with a `page_format_version`
update) or a major bump (rare, requires migration tooling).

**Look for**:
- Changes to constants in `page.rs`: `PAGE_SIZE`, header sizes, magic, layout offsets.
- Changes to serialization in `superblock.rs`, `data_page.rs`, `overflow.rs`,
  `handle_table.rs`, `freemap.rs`.
- New fields in any disk-stored struct.

**Required for any format-affecting change**:
- An entry in `ISSUES.md` documenting the change.
- A new `page_format_version` value if the change is page-local.
- A new MAJOR or MINOR if the change is file-wide.
- Read-side dispatch on the version byte for backward compatibility.
- Migration story (typically lazy: read old, write new).

### 2.2 Reserved bytes

**Why**: Reserved bytes (8..16 in non-superblock pages, 312..8184 in superblocks) are
the forward-compatibility budget. Spending them is a one-way decision.

**Look for**: any code that reads or writes a reserved range.

If the PR uses reserved bytes, the spend must be documented and approved via
`ISSUES.md` like any other format change.

### 2.3 The `PageType` reservation

**Why**: `PageType = 0x00` is reserved so a zeroed page cannot masquerade as a valid
type. Removing this would re-enable a class of bugs.

**Look for**: new `PageType` variants or any code that assigns 0x00 a meaning.

---

## Section 3: Layer dependency graph

### 3.1 Upward references

**Why**: The strict bottom-up dependency graph is what lets you read the codebase in
order without forward references. Adding an upward reference (a lower layer importing
from a higher one) breaks this.

**Look for**: a `use` statement where the importer is at a lower layer than the
imported. Specifically, any `use crate::transaction::*` or
`use crate::handle_table::*` from a layer below 5.

### 3.2 Sideways references

**Why**: Modules at the same layer should not depend on each other. Doing so creates
an implicit hierarchy that contradicts the explicit layer model.

**Look for**: `data_page` calling into `overflow`, `freemap` calling into `data_page`,
etc. The orchestration layer (`transaction.rs`) is where these compose.

### 3.3 New module placement

**Why**: New modules should fit into the existing layer structure cleanly. Adding a
module that "kind of belongs at layer 4 but uses something from layer 6" indicates
the layer model is being violated.

**Look for**: PRs that add new modules. Each should declare its layer (in a doc
comment or `CLAUDE.md` update) and depend only on layers below.

---

## Section 4: Performance correctness

### 4.1 Allocation in the cache-hit read path

**Why**: Reads are the most-called operation. Allocation per read shows up under
profiling as the dominant cost.

**Look for**: new `Vec`, `String`, `Box`, `format!`, `to_owned`, `to_vec` in:
- `read()` and `read_into()`
- `lookup()` in handle_table
- `get_value()` in data_page
- `cache.get()`

The current API contract requires `read` to return a `Vec<u8>`, so one allocation per
call is unavoidable. Multiple allocations per call are a regression.

### 4.2 Syscalls outside `page_io.rs`

**Why**: `page_io.rs` is the only module that should touch the filesystem. Putting
syscalls elsewhere violates the layer model and makes the code harder to mock.

**Look for**: `std::fs::*`, `std::io::*` usages outside `page_io.rs`. Time-related
syscalls (`Instant::now`, `SystemTime::now`) are usually fine; file-related are not.

### 4.3 The hot path checklist

For any change to `read()`, `allocate()`, `update()`, `delete()`, or `commit()`:

- Is there a benchmark before and after?
- Is the in-memory backend numbers ≤ 5% changed?
- Is the file backend numbers ≤ 5% changed?
- If either changed > 5%, is there a clear architectural reason?

If a "performance improvement" PR doesn't include benches, ask for them. If a
"refactor" PR includes a > 5% perf change, that's a hidden behavior change.

### 4.4 New unsafe

**Why**: Chisel uses `unsafe` sparingly and intentionally (mostly for `flock` via
`libc`). New unsafe at any layer ≤ transaction needs careful review.

**Look for**: new `unsafe` blocks. Each should have:
- A safety comment explaining the invariants.
- A test that exercises the boundary condition.
- Justification for why safe Rust isn't sufficient.

`unsafe` for performance reasons (e.g., `get_unchecked` on slices) requires evidence
that the safe version is genuinely a bottleneck. Most claims of "we need unsafe for
performance" are bounds-check elision questions and the right answer is to refactor
to use iterators (which elide checks via `Iterator` trait).

---

## Section 5: Error semantics

### 5.1 Operational vs fatal classification

**Why**: `is_fatal()` drives the poison model. An error that should poison must be
fatal; an error that's caller-recoverable must be operational. Misclassification breaks
recovery.

**Look for**: new `ChiselError` variants. Each must:
- Be classified explicitly in `is_fatal()`.
- Match the existing pattern: state-corrupting → fatal, caller-mistake → operational.

### 5.2 Error path allocations

**Why**: Errors constructed in hot paths (e.g., `InvalidHandle` from a bad lookup)
should be cheap. Allocating a string per error in a tight loop is a common antipattern.

**Look for**: `format!` or `String::from(format!(...))` in error construction. Use
enum variants with structured data instead.

**Pattern**:
```rust
// BAD
return Err(ChiselError::Custom(format!("invalid handle {}", h)));

// GOOD
return Err(ChiselError::InvalidHandle { handle: h });
```

### 5.3 Error message stability

**Why**: Some callers (especially tests, possibly downstream users) match on error
message strings. Changing them silently breaks consumers.

**Look for**: changes to `Display` impls of `ChiselError` variants.

---

## Section 6: Testing

### 6.1 In-memory tests

**Why**: In-memory mode is fast and isolated. Default to it unless the test
specifically exercises filesystem behavior.

**Look for**: tests using `tempfile::NamedTempFile` when `Chisel::open_in_memory()`
would suffice.

### 6.2 Crash-recovery tests

**Why**: The recovery path is what shadow paging buys. Changes to commit, recovery,
or superblock selection need crash-recovery tests.

**Look for**: changes to commit/recovery without a corresponding test that simulates
a crash (typically: open, write, drop without commit, reopen, verify).

### 6.3 Poison-and-recover tests

**Why**: The poison model is mandatory but easy to get wrong. New error paths should
have poison tests.

**Look for**: a fatal error path without a test that:
- Triggers the error
- Confirms `is_poisoned()` returns true
- Confirms subsequent calls return `Poisoned`
- Reopens and confirms the database is at its last-durable state

---

## Section 7: Documentation

### 7.1 ISSUES.md updates

**Why**: ISSUES.md is the decision log. Performance PRs should add an entry with date,
rationale, before/after numbers, and trade-offs.

**Look for**: the "ISSUES.md" file in the diff. If absent on a performance PR,
something's missing.

### 7.2 ARCHITECTURE.md updates

**Why**: Living architecture doc. Changes that affect the layer model, the commit
protocol, or the on-disk format should update it.

**Look for**: `ARCHITECTURE.md` in the diff for layer-affecting or format-affecting
changes.

### 7.3 Comment quality

**Why**: Per CLAUDE.md, comments explain why, not what. New code that violates this
is a documentation regression.

**Look for**: comments that paraphrase the code below them ("increment counter") vs.
comments that explain the design choice ("counter wraps at u64::MAX; safe because
handles are never reused").

---

## Quick-pass summary

For a performance-tagged PR, the minimum review:

1. ✓ Benchmarks before and after, both backends
2. ✓ No format-affecting changes (or properly versioned)
3. ✓ No commit-protocol reordering
4. ✓ Layer graph respected
5. ✓ No new unsafe (or justified)
6. ✓ Poison model preserved
7. ✓ ISSUES.md entry
8. ✓ Tests for the changed code path

A PR missing any of these gets a request for the missing item, not a merge.
