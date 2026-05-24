---
name: guc-series
description: Draft entries in the "All Your GUCs in a Row" blog series for thebuild.com — an alphabetical, one-post-per-parameter tour of every PostgreSQL GUC. Trigger whenever the user asks for the next GUC post, names a specific parameter to write up for the series, says "GUCs in a row" / "the GUC series" / "next GUC," or hands over a parameter name (e.g. "now do `work_mem`") in the context of this series. Also trigger when combining several related parameters into one post (a "double-header" or "triple-header"). This skill specializes thebuild-voice for the series' specific structure, anatomy, and recurring devices; load thebuild-voice as well for the underlying prose voice. Do NOT use for one-off blog posts unrelated to the GUC series, client reports, or code.
---

# All Your GUCs in a Row

A first-draft skill for the GUC blog series on thebuild.com. The series is an alphabetical tour of every PostgreSQL GUC, one parameter per post (related parameters combined), each post short, opinionated, and operationally useful.

This skill assumes **thebuild-voice** and **humanize** and adds series-specific structure. Load both. Everything about *how the prose sounds* (authority, dry wit, unhedged recommendations, cadence, no throat-clearing, no summary endings) lives in thebuild-voice and is not repeated here. This file is about *what a GUC-series post is*.

## Before writing: verify, don't remember

Defaults, contexts, ranges, and behaviors change across major versions, and the model's training data is frequently wrong or stale on specifics. **Search or check the docs before drafting any post.** Confirm against the current PostgreSQL documentation (and pgPedia, which is reliable for per-parameter default/context/version history). In particular verify: the default value, the context, the range/units, the version a parameter or its current default was introduced, and any version-specific behavior change. A post that states a wrong default is worse than no post.

The current stable major version is PostgreSQL 18 (18.3, February 2026). Supported versions are 14–18. Write for that world.

## Format

- **Title:** `# All Your GUCs in a Row: \`parameter_name\``
- **Multi-header:** `# All Your GUCs in a Row: \`first\` and \`second\`` (or `, ` lists for triples)
- **File:** one markdown file per post; `<parameter>.md`, or a combined slug for multi-headers (e.g. `autovacuum_analyze.md`, `bgwriter_lru.md`).
- **Length:** ~400 words is the center of gravity, not a rule. A genuinely trivial parameter gets 250 and stops; one that needs a subsystem explanation gets 700+. **Never pad to hit a number.** A short post that says everything is better than a long one that repeats itself.

## What every post must establish

Early, and without ceremony: the **default value**, the **context** (what's required to change it), and where relevant the **range** and **units**. Weave these into prose or state them in a tight cluster near the top — don't bury them and don't stretch them into a table unless there's genuinely tabular data.

The four contexts, with the operational fact that matters for each:

- **`user`** — any role can set it per-session; usually also per-role and per-database. A per-client concern.
- **`superuser`** — settable live, but only by a superuser (or a role with SET privilege on it).
- **`sighup`** — changed in `postgresql.conf` + reload (`pg_reload_conf()` / `SIGHUP`). No restart.
- **`postmaster`** — **requires a restart.** Call this out plainly; it's the fact that surprises people in an emergency, and "you cannot change this without a restart" is often the single most important sentence in the post.

## The arc

thebuild-voice's arc applies: **problem → why it matters → what to do.** For a GUC post that usually means:

1. What the parameter is and what it controls (mechanically — what actually happens).
2. The failure mode or situation it addresses, or why it exists at all.
3. When to change it, when to leave it, and the concrete recommendation.

The recommendation is **woven into the close as the natural end of the argument** — not appended as a labeled summary. End where the argument ends.

## Structural discipline (read this — it's where drift happens)

Over a long series the posts tend to calcify into a fixed skeleton. Resist it. Specific failure modes that crept in and must be avoided:

- **No fixed four-header skeleton.** Don't stamp every post with the same H2 sections. Many posts need *zero* headers — they're 400 words of prose and headers would be noise. Use a header only when the post is long enough that a reader needs a signpost, and let the header be a real signpost ("The part that actually matters") or a dry punchline, never a generic category label ("Tuning," "Overview," "When to change it") repeated post over post.
- **No bolted-on `Recommendation:` coda.** A bold "**Recommendation:**" paragraph at the end that restates the post is exactly the summary ending thebuild-voice forbids. State the recommendation as part of the prose, plainly, and stop. (The series' best closings read like: "The default is almost always correct. If you're changing it, you probably have a different problem.")
- **Bullets enumerate; prose reasons.** A list of 2–4 genuinely parallel options or scenarios can be bullets. Reasoning chopped into bullets to look organized is a smell — write it as prose. If every post has a three-bullet "reasons" list, that's the tell.
- **Vary cadence per thebuild-voice.** A long sentence carrying a mechanism, then a short one landing the point. Don't let the post settle into a uniform expository pulse.

If a draft has: four same-named headers, a bulleted middle, and a bold Recommendation coda — it has drifted. Rewrite it as prose with at most one or two earned headers and a woven close.

## Recurring devices that work

Use these where they fit. **Do not formularize them** — a device used every post stops being a device and becomes scaffolding.

- **Reframe the parameter as what it actually is.** The strongest posts relabel the knob: `archive_timeout` is an RPO dial, not an archive-plumbing setting; `authentication_timeout` is slot-occupation defense, not an auth-duration knob. Find the operational truth the docs' framing obscures.
- **The "tour."** When a parameter can't be understood without its subsystem, explain the subsystem **once**, in the first post of the cluster, then have sibling posts reference back rather than re-explain. Established tours so far: the **writeback tour** (`backend_flush_after`), the **checkpoint tour** (`checkpoint_timeout`/`completion_target`), the **tuple-freezing tour** (`autovacuum_freeze_max_age`), the **multixact tour** (`autovacuum_multixact_freeze_max_age`). New clusters (WAL, planner, logging) will want their own. Cross-link to the tour; keep the siblings short.
- **The vestigial-parameter through-line.** Some parameters survive only because PostgreSQL doesn't break things: `array_nulls`, `backslash_quote`, `bonjour`, `bytea_output`, the `SQL_ASCII` angle of `client_encoding`. When you hit another, name the pattern and cross-link the family. The tone is affectionate-exasperated, not contemptuous.
- **Vintage defaults.** Many defaults are calibrated for 2008-era hardware (the bgwriter, checkpoint, and vacuum-cost defaults especially). Say so when raising them is the right call. **Use fresh phrasing every time** — "calibrated for 2008," "hardware built since the Obama administration," etc. The beat "After fifteen years." has been used twice already (in the autovacuum cluster) and is retired; don't reuse it.
- **One-line credential asides.** Real PGX consulting experience can ground a claim ("I have diagnosed an outage with exactly this root cause; it is not theoretical"). One clause, not a war story. Never make the post about the anecdote.
- **Ground diagnostics in real instrumentation.** Name the actual view/metric/catalog to look at, not "monitor it": `pg_stat_bgwriter.maxwritten_clean`, `n_tup_hot_upd`, `age(relfrozenxid)`, `pg_stat_progress_vacuum.index_vacuum_count`, `checkpoints_req` vs `checkpoints_timed`. This is what makes a post usable instead of merely correct.

## Cross-linking and series cohesion

- Link to earlier posts by relative slug where it genuinely helps the reasoning (`[\`backend_flush_after\`](./backend_flush_after)`).
- Set up natural pairs when a parameter's sibling lives in a later cluster — e.g. `client_min_messages` explains the client-vs-log split and points at `log_min_messages`. This is *reasoning* ("if you want the log, that's the other parameter"), not foreshadowing.
- **Do not tease the next post.** thebuild-voice forbids foreshadowing; the posts end on their own argument. ("Next up: X" is fine in chat with the author; it never appears in a post.)
- When you discover an error in an already-drafted post, fix it via an honest **errata** at the top of the relevant later post and flag it to the author — the series corrects itself in the open rather than silently.

## Technical and copyright standards

Inherited from thebuild-voice, restated because they're load-bearing here:

- Exact version numbers, exact GUC names, exact syntax. `pgvector`, `pg_stat_statements`, `sync_file_range()` — canonical spelling always.
- Note version-specific behavior explicitly and give version-aware advice when the right answer differs across supported versions (e.g. the PG 17 TIDStore change to `autovacuum_work_mem`, the PG 18 `autovacuum_vacuum_max_threshold`).
- **At most one short docs quote per post**, under 15 words, in quotation marks, only when the docs' own wording earns it (the rare moment the docs editorialize). Otherwise paraphrase. Default to paraphrase.

## Pre-submission checklist

- [ ] Verified default, context, range, and version facts against current docs — not from memory.
- [ ] First sentence is specific and earns attention; no throat-clearing.
- [ ] The parameter's mechanics are actually explained, not just named.
- [ ] There's a concrete, unhedged recommendation, woven into the close — not a bolted-on summary.
- [ ] Structure is prose-first: no rote four-header skeleton, no bulleted-reasoning, no `Recommendation:` coda.
- [ ] Cadence varies; at least one mechanism-carrying long sentence and at least one short landing.
- [ ] Diagnostics name real views/metrics where relevant.
- [ ] Cross-links are accurate; no next-post foreshadowing inside the post.
- [ ] Length fits the parameter — short if trivial, long only if it earns it. Nothing padded.
- [ ] One docs quote maximum, under 15 words; everything else paraphrased.

## Micro-example (opening + close, prose-first)

Opening that reframes and skips preamble:

> `archive_timeout` is an RPO knob wearing an archiving costume. Set it, and PostgreSQL forces a WAL segment switch after the specified interval even if the segment isn't full — which means the question it actually answers is "how much data am I willing to lose if the primary dies right now?"

Close that ends on the argument, no summary:

> If your tolerance is under a minute, this isn't your parameter — you want a replica, not a faster archive. Set it to `60s`, and spend the energy you saved on `synchronous_commit`.

## Reference

See `references/examples.md` for a fuller set of representative openings, default/context clusters, tour transitions, vestigial-parameter framings, grounded diagnostics, and closings drawn from posts already in the series. Consult it when calibrating — especially for openings and closings, where drift toward generic structure is most likely.
