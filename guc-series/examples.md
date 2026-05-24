# All Your GUCs in a Row — Worked Examples

Calibration anchors drawn from posts already published in the series. Read these before drafting when you need to feel the shape of a post — especially openings (where drift toward generic tech-blog prose is most likely) and closings (where the bolted-on summary tends to creep back in).

These complement thebuild-voice's own `examples.md`. That file calibrates the *voice*; this one calibrates the *series-specific moves* — reframings, tours, the vestigial through-line, and prose-first structure.

---

## Openings

**Reframe the parameter as what it actually is.** The strongest series openers relabel the knob in the first sentence.

> `archive_timeout` is an RPO knob wearing an archiving costume.

> `authentication_timeout` is a connection-slot defense that happens to be spelled like an authentication setting.

What works: the reader's mental model of the parameter is corrected before they've finished the first line. No "this parameter controls…".

**Lead with the political or historical reason it exists.**

> `allow_alter_system` is one of those parameters that exists because the PostgreSQL community had an argument and someone had to win.

> `backslash_quote` is a parameter whose entire reason for existing is a 2006 security disclosure about multibyte encodings, and which almost nobody has needed to think about since.

What works: immediate specificity, dry framing, no preamble explaining what the parameter category is.

**Name the failure mode in the first breath.**

> Turn off `autovacuum` and you will eventually have a bad day. The only variables are which bad day, and how bad.

What works: states the stakes flatly, sets up the post as a tour of consequences, no hedging.

**For a tour-opening post, signal the subsystem up front.**

> `backend_flush_after` is the first of four parameters that all do the same thing to four different writers, so we'll spend most of this post on the thing they do and the rest of the cluster can be short.

What works: tells the reader this post carries the subsystem explanation, justifies its length, sets up the cross-references.

---

## Stating the default/context cluster

Tight, woven, no ceremony. Good:

> The default is `NOTICE`, the context is `user`, and the levels run in increasing severity from `DEBUG5` up to `ERROR`.

> Default `0`, meaning disabled. Context `sighup`, so you can turn it on without a restart — which matters, because the reason you'd want it tends to arrive unannounced.

Note the second one slips the operational consequence of the context (`sighup` = no restart) into the same sentence. That's the move: don't just state the context, state what it buys you.

Bad (don't do this):

> | Setting | Value |
> |---|---|
> | Default | NOTICE |
> | Context | user |
> | Min | — |

A table for three scalar facts is scaffolding. Use prose unless the data is genuinely tabular (e.g. comparing the defaults of four sibling parameters at once, which *is* a table worth having).

---

## Calling out `postmaster` context

When a parameter needs a restart, that's frequently the most important sentence in the post. Say it plainly.

> Context is `postmaster`, which means changing it requires a full restart. You cannot raise this in the middle of the incident that makes you wish it were higher — which is, of course, exactly when you'll want to.

---

## The "tour" (subsystem explained once, referenced after)

Tour-bearing post introduces the machinery:

> A checkpoint is the point at which PostgreSQL guarantees every dirty page from before that moment is on disk, so that crash recovery never has to replay WAL older than the last completed checkpoint. Everything in this cluster is about making that periodic event cheap and smooth instead of a stall you can see in your latency graphs.

Sibling post leans on it instead of repeating:

> `checkpoint_flush_after` is a writeback hint, the same mechanism covered in [`backend_flush_after`](./backend_flush_after): tell the kernel to start flushing dirty pages now, in bounded chunks, rather than letting them pile up for one enormous `fsync` at checkpoint time.

What works: the sibling spends one sentence and a cross-link where it would otherwise spend three paragraphs.

---

## The vestigial-parameter through-line

> `bytea_output` joins the small club of parameters that survive only because PostgreSQL refuses to break anything: `array_nulls`, `backslash_quote`, `bonjour`. The escape format isn't *wrong*, exactly. It's just that nobody has had a good reason to choose it since `hex` arrived in 9.0.

Affectionate-exasperated, not contemptuous. Names the family and cross-links it.

---

## Vintage-defaults beat (fresh phrasing each time)

> The default of `100` was a sensible number for the spinning disks of 2008. On an NVMe array it is almost insultingly conservative.

> These defaults assume a machine that no longer exists. Raise them.

Do **not** reuse "After fifteen years." — it's been spent twice already. Find a new way to say the hardware moved on each time.

---

## Diagnostics grounded in real instrumentation

Vague (avoid):

> Monitor your system and adjust if the background writer isn't keeping up.

Grounded (do this):

> Watch `pg_stat_bgwriter.maxwritten_clean`. If it's climbing, the background writer is hitting `bgwriter_lru_maxpages` and stopping before it's done — it wants to write more and you're not letting it.

The grounded version names the metric, the threshold behavior, and what a rising value *means*. That's the difference between a post that's correct and a post that's usable.

---

## One-line credential aside

> Synchronous-commit pile-ups under load are not hypothetical; I have watched one take down a production primary while every backend sat waiting on a standby that had quietly fallen behind.

One clause of authority, then back to the parameter. The anecdote serves the point; it never becomes the point.

---

## Closings (end on the argument — never summarize)

> The default is almost always right. If you find yourself reaching for this one, the interesting question is what problem you think you're solving, because it probably isn't this.

> Leave it on. The cost is a rounding error and the failure it prevents is a pager at 3 a.m.

> If your tolerance is under a minute, this isn't your parameter — you want a replica, not a faster archive. Set it to `60s` and move on.

> So: `hex`, always, and the only reason to know `escape` exists is so you recognize it in someone else's dump file.

What works in all four: the post stops the instant the argument is complete. No "In summary." No bold "Recommendation:" header. No "in the next post." The last sentence is load-bearing, not a recap.

---

## Anti-patterns specific to the series

Beyond thebuild-voice's general list, watch for these series-specific tells of a drifted draft:

- A bold **`Recommendation:`** paragraph at the end. Delete it; weave the recommendation into the close.
- The same H2 headers ("Overview," "Tuning," "When to change it," "When to leave it") on post after post. Use earned signposts or none.
- A three-bullet "reasons to change it / reasons not to" block that's really reasoning in disguise. Write it as prose.
- Re-explaining a subsystem a previous post already toured. Cross-link instead.
- Padding a trivial parameter to hit ~400 words. Some posts are 250 words and finished.
- Stating a default or context from memory. Verify first; the docs are one search away and training data is often stale.
