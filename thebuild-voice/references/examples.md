# thebuild.com Voice — Worked Examples

Representative passages for calibration. Read these before drafting when you need to anchor the voice — especially for openings, which are where drift toward generic tech-blog prose is most likely.

---

## Openings

**Series opener (blog):**

> Welcome to what is either an ambitious project or a cry for help, depending on your perspective: a complete tour of every PostgreSQL GUC, in alphabetical order, with enough detail to actually be useful.

What works: self-aware framing, no "in this post we will," establishes scope without explaining what a GUC is.

**Problem-first (blog):**

> `allow_alter_system` is one of those parameters that exists because the PostgreSQL community had an argument and someone had to win.

What works: immediate specificity (the parameter name), dry framing of a political story, no preamble.

**Report opener (standard register):**

> Given your current operational posture — nightly vacuums completing reliably, dedicated instances per table, manual reindex windows — postponing partitioning is reasonable.

What works: leads with the recommendation, grounds it in the client's specific situation, no "Thank you for sharing the document."

---

## Stating a recommendation

Good:

> Do this. UUIDv1 gives you time-ordering but the bit layout is hostile to B-tree locality. UUIDv7 fixes that. Expect measurably less index page splitting on insert-heavy tables. Easy win.

Note the two-word opening sentence, the short punchy follow-ups, and the no-nonsense "Easy win" close. No "you might want to consider."

Bad (for contrast):

> You might want to consider potentially adopting UUIDv7, as it could offer some benefits over UUIDv1 in certain scenarios involving B-tree indexes.

---

## Fragment used correctly (rare, emphatic)

> The `REINDEX CONCURRENTLY` will take the better part of a day with aggressive IOPS provisioning. Plan for a quiet weekend, not a 12-hour window.
>
> Really.

The "Really." lands because the surrounding prose is not full of fragments.

---

## Parenthetical aside

> The `max_connections` default of 100 is fine for development (and for people who have never seen what happens when 100 connections each try to sort a million rows in `work_mem`).

The aside is where the voice shows up — a dry observation that presumes shared experience.

---

## Cookbook format (multiple scenarios)

> ### If you're on RDS
>
> You can't set `shared_preload_libraries` outside the parameter group, and the parameter group change requires a reboot. Plan accordingly.
>
> ### If you're self-hosted
>
> Edit `postgresql.conf`, restart, move on with your life.
>
> ### If you're on Aurora
>
> My condolences. The parameter group behavior is similar to RDS but with additional surprises around cluster vs. instance parameters. Test first.

Each scenario is short. Distinct cases get distinct headers. No introductory paragraph explaining that different environments behave differently.

---

## Ending (blog)

> The default value is almost always correct. If you find yourself changing it, you probably have a different problem.

No summary. No "In conclusion." No "Hope this was helpful!" The piece ends where the argument ends.

---

## Ending (report)

> The staging-table approach is the most interesting idea in the document. Collapsing 4–6 UPDATEs into 1 materially reduces dead tuple generation, index churn, and WAL volume. I'd prototype this first — the ROI here may exceed partitioning.

Ends on the recommendation. No "Please let me know if you have questions."

---

## Anti-patterns to avoid

These are phrasings that would feel immediately wrong in the voice:

- "In this post, we'll explore…"
- "It's important to note that…"
- "There are several things to consider…"
- "Hopefully this was helpful!"
- "Let me know if you have any questions."
- "This is a really exciting feature that…"
- "You might want to think about…"
- "It could be worth considering…"
- "Stay tuned for the next post where…"

If a sentence from your draft would fit comfortably into a mid-tier SaaS company's engineering blog, rewrite it.
