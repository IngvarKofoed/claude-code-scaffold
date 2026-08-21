# Per-entry changelog files

Retire the scaffold skill's single-file changelog. A repo's changelog becomes a folder,
`docs/changelog/`, holding one file per entry named `YYYY-MM-DD-<slug>.md` — permanently, with
no fold, no archive file, and no numbers. Root `CLAUDE.md`'s required reading stops loading the
whole changelog and instead loads a curated `docs/CHANGELOG-DIGEST.md` plus a bounded window of
the folder. The staging area the skill currently describes (`docs/changelog.d/` fragments, the
fold at landing, the empty-on-`main` invariant, the janitor rule, the residual renumber) is
deleted rather than adapted: the fragment file *is already the entry file*, and every piece of
that machinery exists only to convert it into a line range in another file.

## Amendment, 2026-08-21 — the digest is removed

**Everything below about `docs/CHANGELOG-DIGEST.md` is superseded.** The digest is gone: there is
no curated file, and required reading is now `@docs/CONCEPT.md` + `@docs/ARCHITECTURE.md` plus a
three-step changelog read — the **full** `ls docs/changelog` index (not `| tail -40`), the newest
10 entries in full, and a `grep -rli` over the folder before touching an area.

*Why.* This spec accepted the digest as "the one shared append-target" on three grounds — a line is
earned by only ~1 change in 10, the conflict is loud, and appends resolve by keeping both sides in
either order (see *The digest is the one shared append-target…* below). Those grounds hold for the
**append**. They were never argued for the other operation the same rule mandates: *"when the digest
passes ~100 lines, tighten its older half."* That is a rewrite of shared lines, not an append — a
real semantic conflict with no keep-both resolution — and it falls due precisely when the project is
busy enough to have parallel work in flight. So the design's one hard-conflict operation was being
paid to save ~1.5k tokens at 100 lines, against the 64k the old single-file scheme cost. The
per-entry folder existed to remove exactly this failure class; leaving one file in it that every
milestone appends to and every compaction rewrites was the last instance of the thing being removed.

*Priced honestly, this is not a token saving.* Dropping ~100 curated lines while unbounding the
index trades a fixed ~1.5k for roughly a dozen tokens per entry, growing for the life of the repo —
at 537 entries the full index already costs more than the digest it replaced, and a long-lived repo
pays a large multiple of it. What the change buys is **conflict-freedom and the end of a lossy
compaction step**, not cheaper session starts. The template's ~1000-entry crossover — past which the
full listing moves into the delegated subagent's brief and the inline read becomes a tail — is what
keeps the cost from growing without limit.

*What carries the reach instead.* The digest's job splits three ways, none of them a shared file:
the **full index** names every entry ever (one cheap line each, and a derived listing, so zero
contention); the **window** gives depth on recent work; the **grep** opens anything the index names.
The load moves onto the slug — now required to name the thing rather than the activity, since it is
the only part of an entry every future session sees — and onto the back-citation rule already in
*Changelog discipline*, which becomes the sole mechanism carrying rationale forward.

*The accepted cost.* Nothing is guaranteed to be *read* past the window any more, so a decision from
six months ago is reachable only if a session thinks to grep for it. That is a real regression
against the digest and it is taken knowingly: the mitigation is that the grep is mandated rather than
suggested, and that the index makes the search terms visible instead of guessed.

*Migration.* A repo already carrying a digest converts rather than freezes — each line folds into the
`## Detail` of the entry it names (the one sanctioned edit to a landed entry, since it adds at the
existing address), lines resolving to no entry are reported rather than dropped, and only then is the
file deleted. This lives in `references/audit-checklist.md` as **R1's migration**.

*What partly buys the reach back — an ephemeral digest.* Required reading gains an **optional**
step between the window and the grep: delegate a digest of the entries behind the window to a cheap
subagent (Haiku or equivalent), which returns ~20 lines of what must not be re-litigated, and
**never persist it**. This is not the rejected generated index. That rejection — *"a generated index
is a fold step by another name and goes stale silently"* — turns entirely on the artifact being
**committed**; staleness requires persistence, so a summary rebuilt from the entries every session
cannot drift from them, and nothing in git can contend for a file that is never written. Delegating
to a cheap model is what makes a 50-entry read affordable at session start: the reading lands in the
subagent's context and only the 20 lines reach the session's. The residual cost is nondeterminism —
a human curated the committed digest, whereas a model re-decides each session and can omit what
mattered — which is why the template fixes the *class* to surface rather than asking for a summary.
It is optional because it buys nothing in a repo whose window already covers everything.

*Also rejected here:* deriving a **committed** digest from per-entry markers, on this spec's own
no-generated-index grounds (*Alternatives considered* → "A generated `RECENT.md`"); and a
`docs/digest/` folder of milestone files, which trades the hot file for a second folder with
overlapping semantics, a second path decision per milestone change, and nowhere for compaction to go.

## Key decisions

- **Per-entry folder is the permanent home** (diverges). `docs/changelog/YYYY-MM-DD-<slug>.md`,
  written directly wherever `HEAD` is. Retires `docs/CHANGELOG.md` as a write target and
  `docs/CHANGELOG-archive.md` as a concept. Two branches cannot touch the same path, so the
  merge conflict and the numbering collision both stop existing.
- **Date-slug filenames; the number space is closed** (breaking). New entries have no number and
  are cited by slug path. Existing `entry N` citations keep resolving because every backfilled file
  records the number it came from — the two address spaces cannot collide, because no new number is
  ever issued. Resolved in *Alternatives considered* against a numeric prefix.
- **Reading is three tiers, not one** (new). `@docs/CHANGELOG-DIGEST.md` always; then
  `ls docs/changelog | tail -40` as an index; then the newest 10 files in full plus any the index
  shows bear on the work at hand. The listing is load-bearing, not decorative — date-slug names
  are self-describing, so 40 filenames are a table of contents for ~400 tokens.
- **`docs/CHANGELOG-DIGEST.md` keeps the distill half of the archival pass** (extends). Curated
  decisions and milestones, hand-maintained, stays in required reading. The renumber-and-move
  half of the old archival pass is retired.
- **Entries get a two-tier shape** (extends). A required lead bounded exactly as today — 1–5
  lines, ~20 words per line — plus an optional detail body under a purpose test and a ~40-line
  ceiling. The bound now protects the read surface rather than the whole file.
- **Changelog discipline becomes mode-independent** (new). Nothing about the entry depends on the
  git mode, the branch, or the worktree. This deletes step 4's fragment-bullet paragraph and the
  two trim clauses that pointed at it, and removes the changelog from both landing sequences.
- **`git merge --no-commit main` is retired from the worktree landing sequence** (diverges). Its
  only stated justification was keeping the fold inside one commit. With nothing to fold, a clean
  merge may commit itself; that merge commit carries no authored content and needs no review.
- **Migration is evidence-triggered, all-or-nothing, and it backfills every existing entry**
  (new). A young single-file repo audits **OK**. Where the pressure is measurable, the migration is
  one item spanning R1, R3, R6/R6b, the distillation into the digest, **and a full split of the
  existing changelog into per-entry files** — because a migration that leaves the folder empty has
  taken 64k of history out of required reading and given nothing back. Dates for backfilled entries
  are synthesized, not recovered; see *The backfill* below for why, and for the two things that
  forces (a number marker in each file, and a verification gate before the originals are removed).
- **One convention, universally** (new). No young-repo variant. The folder's read policy
  degenerates to the old behaviour at small N for free; the single file does not degenerate at
  large N, it degrades, and the exit costs a migration.

## Goals

- Cut the unconditional session-start cost of the changelog from ~64k tokens (measured on
  `/Users/ingvar/work/flux`) to a bounded, curated read that still reaches distant history — the
  digest is what carries the reach, which is why a migration populates it rather than leaving it
  empty.
- Remove the numbering invariant, which has failed four times in 537 entries and cannot hold in a
  repo with worktrees or concurrent PRs — there is no global sequencer.
- Remove the recurring archival toil (six passes in flux, one per 70–100 entries).
- Leave every existing `entry N` citation resolving. There are ~390 such citations outside the
  changelogs in flux (a raw grep returns 716, double-counting a worktree copy of `docs/`);
  numbering is a load-bearing addressing scheme and cannot simply be dropped.
- Leave no file in the skill presenting both schemes as current, and no landing step with nothing
  to fold. The rubric must still *describe* the retired scheme in order to detect it — that is its
  job, and it is not a violation of this goal.

## Non-goals

- Rewriting git history. The pre-migration changelog stays reachable forever through git; only the
  working tree changes.
- Renumbering or reordering entries. A backfilled file keeps the number it had, recorded in its body.
- Making every previously-scaffolded repo report a finding.
- Changing what an entry *says*. The durable-project-memory rules survive verbatim.
- Sharding `docs/changelog/` by year or month. `ls | tail -40` over 537 files is fine, and a
  shard rule buys nothing.

## Design

### What a scaffolded repo looks like

```
docs/CONCEPT.md
docs/ARCHITECTURE.md
docs/CHANGELOG-DIGEST.md          curated, small, always read
docs/changelog/
  2026-08-18-run-lock-scope.md
  2026-08-19-exclusive-sink-editing.md
  2026-08-19-back-closes-forms.md
```

A migrated repo looks the same. Its older entries carry synthetic dates and an `<!-- entry N -->`
marker recording the number they had; `docs/CHANGELOG.md` and `docs/CHANGELOG-archive.md` are gone
from the working tree and reachable through git.

The digest sits beside the folder rather than inside it for two reasons: a file inside
`docs/changelog/` lands in the `tail` window and displaces a real entry, and the name has to say
it digests *the changelog* rather than reading as a competing decision log.

### The entry file

Filename `YYYY-MM-DD-<slug>.md`, dated when the entry is written, slug 2–4 kebab-case words —
deliberately the same convention as `docs/specs/`, which these repos already run successfully.

The body is two tiers:

- **Lead** — an `#` heading plus one paragraph. Bounded exactly as the current template bounds an
  entry: 1–5 lines, ~20 words per line at most, never one packed run-on line. Same content rule,
  unchanged: what is now *true that wasn't before*, plus the *why* in a clause where it isn't
  obvious — not a recap of the diff.
- **Detail** — optional, below the lead. Only what a future session must know and cannot get from
  the diff: the alternative that was rejected and why, a non-obvious constraint, a known limit, a
  deferred follow-up, what the verification did and didn't cover. It never restates the lead.
  Ceiling ~40 lines; past that the change wanted a spec, so write one and link it.

The lead carries the bound because the lead is the only part read in bulk. This is what flux
entries already do — entry 533 runs 50 lines but opens with a compliant 6-line lead — so the rule
formalizes observed practice instead of restating a rule that has been violated tenfold. Say so
in the template's voice: *the lead is bounded because it is what a session reads to orient; the
detail is bounded by purpose because nothing else bounds it now that entries no longer share a
file.*

Cross-references between entries use the slug (`2026-08-19-exclusive-sink-editing`), the way flux
entry 534 already refers to 533. `entry N` remains a valid citation for entries that predate the
migration, resolved by grepping the folder for their marker; no new entry ever gets a number.

**A landed entry file is never renamed, moved, or deleted.** Its path is its address, and this is
the invariant that replaces *never reuse or reorder numbers* — the template must state it, because
it is the one silent failure the layout does not remove on its own. A duplicate path is blocked by
git; a rename is not, and it breaks every citation to that entry without producing a conflict.

### The read policy

Root `CLAUDE.md`'s *Always read first* becomes:

- `@docs/CONCEPT.md`, `@docs/ARCHITECTURE.md` — unchanged.
- `@docs/CHANGELOG-DIGEST.md` — curated decisions and milestones.
- Then, as an instruction rather than an `@`-reference: `ls docs/changelog | tail -40` for the
  index, read the newest **10** in full, and read any further entry the index shows bears on what
  you are about to touch. An absent folder means no entries yet — a fresh repo's first entry
  creates it, and the instruction must tolerate that rather than failing on day one.

The window is an instruction because no `@`-reference can express "the newest N of a folder".
That is a genuine cost of the layout and the template should name it rather than paper over it:
the read is something the session performs, not something already in context. State it as a
numbered action so it cannot be read as optional.

It will still be skipped some fraction of the time, and the failure is quiet: the session runs on
the digest alone and re-litigates the last few days with no signal that anything is missing. Under
the old scheme skipping was impossible, so this is a real regression and the template owes it a
mitigation rather than silence — number the action, and require a self-check before an entry is
written (*did I list the folder this session?*). A session-start hook would make it deterministic;
note that as the option it is and leave it there, since hooks are out of this skill's scope.

Why the index tier exists rather than a deeper full read: flux's ten most recent entries span 281
lines, so ~28 lines each. Twenty read in full is ~22k tokens — a 3× improvement on 64k, which is
not the order of magnitude the change should buy. Ten is ~11k, and the index adds ~400 tokens for
40 entries while telling the session what *else* exists, which the old scheme never did for the
314 archived ones. Session start lands near 13–15k with the digest, against 64k today.

**Why the window alone is insufficient, and what covers the gap.** At ~30 entries in three days,
a 10-deep window is barely a one-day memory, and recent work leans on distant entries constantly —
deciding not to rebuild something already tried and removed. The digest is what covers that, and
it is the piece that must not rot. Rule: a change earns a digest line when it is a milestone or a
decision a future session must not re-litigate; otherwise the entry file alone. One to three
lines per item, pointing at the entry's slug. When the digest passes ~100
lines, tighten its older half rather than growing it — that compression is the surviving half of
the archival pass, kept in the one place it is cheap.

**The digest is the one shared append-target the design leaves, and that is accepted rather than
engineered around.** Two parallel changes both earning a digest line conflict at end-of-file — the
failure class the entry files remove, in miniature. It is tolerable for three stated reasons: a line
is earned only by a milestone or a decision a future session must not re-litigate, so perhaps one
change in ten; the conflict is loud rather than silent; and the resolution is trivial, because the
digest is not sequenced — keep both sides, in either order. The old scheme's defect was that *every*
parallel pair conflicted on a file *every* change had to touch. Say the resolution policy in the
template so nobody invents one. The rejected alternative — write digest lines only from `main` after
landing — reintroduces a mode-dependent rule and, worse, hands the line to a session that no longer
holds the context that earned it.

**Degradation at small N.** A six-entry repo lists all six and reads all six; the digest is
empty and says so. That is the old single-file behaviour, at no extra cost. This is the argument
for one universal convention rather than a threshold.

### What dissolves, and the three things that don't

Deleted outright, with nothing replacing them: `docs/changelog.d/` and the fragment concept; the
fold in both landing sequences; the empty-on-`main` invariant; the janitor rule; the residual
renumber case; monotonic numbering, append-to-the-end and never-reorder; the archive-on-phase-
completion move and global number uniqueness; the seeded numbering header; the "do not create
`docs/changelog.d/`" note; the `--no-commit` requirement.

That the janitor rule needs no replacement is the point worth stating in the rubric: it existed to
catch a silent breach of an invariant that no longer exists. An entry file on `main` is exactly
where it belongs.

Three pieces survive, and the spec is explicit about them because each survives for a *different*
reason than it was written for:

1. **The base-copy reading caveat.** The worktree section's note that the changelog you read on a
   branch is the base copy, missing anything landed since, is about *reading* and still true of a
   folder. Keep it. Everything in that paragraph about *writing* goes.
2. **The all-or-nothing offer mechanism** (step 6's narrowing exception). The mechanism survives;
   its subject changes from the fragment migration to the layout migration.
3. **The entry-quality rules.** Durable project memory, the why in a clause, not a diff recap, the
   bad-vs-good worked example, and a per-line size bound rather than "as short as possible". These
   are layout-independent and survive intact — the example loses its number prefix, nothing else.

### Migration, and the mid-flight branch

The migration converts the whole changelog rather than freezing it: every existing entry becomes a
file in `docs/changelog/`, and `docs/CHANGELOG.md` / `docs/CHANGELOG-archive.md` are removed from
the working tree once the conversion is verified. Nothing is lost — git holds the originals forever,
and `git show <pre-migration-sha>:docs/CHANGELOG.md` retrieves them — and no history is rewritten.
This is a change in kind from a freeze, so it is offered as a conversion and priced as one.

#### The backfill

**Dates are synthesized, not recovered.** Per-entry git archaeology (`git log -S` per entry, or
reconstructing additions from `git log -p`) is fragile: entries added in bulk, edited after the
fact, or moved during an archival pass all give the wrong answer, and there are hundreds of chances
to be wrong. Synthetic dates are wrong in a bounded, uniform, visible way instead of wrong in
hundreds of unpredictable ways.

The rule, calibrated so the backfill spans the repo's actual life rather than inventing history
outside it:

```
span_days = (yesterday - first_commit_date)      # one git call, not one per entry
per_day   = ceil(total_entries / span_days)
```

Then walk entries newest-first, assigning `per_day` of them to yesterday, the next `per_day` to the
day before, and so on. On flux: 537 entries over a 140-day span gives 4 per day across 135 days,
the oldest landing 2026-04-07 against a first commit of 2026-04-01 — inside the repo's life, which
is the property that matters. Where there is no git history to read, ask for a start date rather
than guessing a span.

**What the dates are and aren't.** They order entries correctly and place them in roughly the right
season. They are not the real dates, and in ~46 of flux's entries the filename date will disagree
with a real date written inside the entry (a cited `docs/specs/YYYY-MM-DD-*.md` path). That is
accepted: the filename's job is sort order and rough vintage, the entry's own text stays the
authority on when anything happened, and the pre-migration file is one `git show` away. Record it in
the migration's own digest line so nobody later mistakes these dates for evidence.

**Same-day order is arbitrary, and that is fine.** Four entries share a date and sort by slug. The
window reads all of them; where sequence is load-bearing, the entries say so in their own text.

**Every backfilled file records the number it came from.** Not optional — it is what keeps ~390
external `entry N` citations and 92 in-entry ones resolving once the numbers leave the filenames and
the originals leave the tree. A marker line in the body (`<!-- entry 534 -->`) makes
`grep -rl 'entry 534' docs/changelog/` the resolution path, replaces exactly what the original
file used to provide, and costs one line per file. The three duplicated numbers (516, 517, 519) resolve
to two files each, which is honest: the repo genuinely has two of each.

**Content is copied verbatim, minus the leading number.** The two-tier lead/detail shape governs
*new* entries. Historical ones are not rewritten, re-bounded, or reformatted — the backfill is a
re-filing, not an edit, and 537 rewrites would be 537 chances to change what an entry claims.

**The verification gate, before anything is deleted.** Count entries in the originals
(`grep -cE '^[0-9]+\. ' docs/CHANGELOG.md docs/CHANGELOG-archive.md`) and count the files produced.
They must match, and every distinct entry number must appear in exactly as many marker lines as it
had entries. If the counts disagree, stop and report: keep the originals, leave the folder in place
for inspection, delete nothing. Deletion happens only on a clean match and in the same commit as the
backfill, so the conversion is one reviewable diff and one `git revert` away.

**Cost, stated at the offer.** Reading ~470 KB and emitting ~537 slugged files is the expensive step
by an order of magnitude — roughly 120k tokens of reading plus the writes. Delegate it as a fan-out
over entry ranges, never an inline pass. A user who won't pay it declines the migration and keeps the
single file, which stays a supported state.

#### The rest of the migration

The digest is still curated as part of the same item, required reading is re-pointed (R1), and
*Changelog discipline* plus the landing sequences are rewritten (R3, R6/R6b). What the pointer
paragraph says changes: with the originals gone, it records that entries predating the migration
carry synthetic dates and an `<!-- entry N -->` marker, and that an `entry N` citation resolves by
grepping `docs/changelog/`.

`docs/changelog/` **is** created by this migration — it has hundreds of files to hold. "The first
entry creates it" stays true only for a fresh scaffold.

#### The mid-flight branch

A branch holding an unfolded `docs/changelog.d/<slug>.md` when the convention changes lands it as
`git mv docs/changelog.d/<slug>.md docs/changelog/YYYY-MM-DD-<slug>.md`, dated the landing day — a
real date, since it is a real landing — and then does not fold. The fragment was already an
unnumbered entry body in a per-change file, so nothing is converted, and it needs no number marker
because it never had a number. `docs/changelog.d/` disappears with its last fragment; a fragment
holding several entries splits into one file per entry.

A fragment already **stranded on `main`** is what the deleted janitor rule existed to catch, so the
migration sweeps for it: check `main` for `docs/changelog.d/` and rename anything found into the
folder, dated that day, because nothing will ever look there again.

The rarer case — a branch that wrote a *numbered* entry straight into `docs/CHANGELOG.md` under the
retired convention — is simpler here than under a freeze: the file it edited no longer exists on
`main`, so the merge deletes it, and the branch's entry is carried over as its own dated file with
its number recorded, like any backfilled one.

### Per-file edits

**`templates/CLAUDE-root.md`** — rewrite from scratch: *Always read first* (lines 7–11, gaining
the digest and the window, losing `@docs/CHANGELOG.md`), *Changelog discipline* (15–25) including
the worked bad-vs-good example at 27–34 (which loses its number prefix and gains the lead/detail
shape), the Branch+PR fold paragraphs (113–117), and the worktree fragment/fold paragraphs (131,
137, 139).

Four further references to the retired scheme sit outside those ranges and must go with them, or
the file keeps a fold with nothing to fold: line 119's "or into the fragment where you're on a
branch or in a worktree" — inside a paragraph the template marks as *always stays*, so it is edited
rather than deleted, to "including its changelog entry file"; line 133's "the fold rule in that mode
applies unchanged"; and line 145's "fold per the mode's fold rule". Treat the enumeration as
exhaustive: a `grep -n 'fragment\|fold\|changelog\.d' templates/CLAUDE-root.md` must come back
empty when the edit is done.

The Branch+PR paragraph is not simply deleted: it becomes one short paragraph saying that
*nothing* about the changelog is special at landing — the entry file rides the PR with its change
and merges with it, and no file is shared with another PR, so nothing goes stale while the PR is
open. The absence of a step is worth stating, because an agent carrying generic expectations will
look for one.

Landing step 1 becomes plain `git merge main` — resolve conflicts if any, commit — retaining only
the clause that says the merge is what makes step 2 a fast-forward, plus the new note that a clean
merge commits itself and that merge commit needs no review.

**That note is scoped to the clean case, and the template must say so**, or it reads as blanket
permission to land a hand-resolved merge unreviewed. R6b's existing rule — the resolution sits
inside the reviewed diff — is listed below as untouched, and it still holds for free: a *conflicted*
merge leaves the resolution uncommitted by default, so it passes the review gate before its commit
with no `--no-commit` needed. That is the whole reason the flag can go without weakening the
guarantee, and it is worth one sentence in the template rather than being left to inference. Step 2's "`main` moved" bullet
survives as a git fact with its numbering half stripped: re-merge `main`, commit, retry the
fast-forward. The template must not mention the changelog conflict that used to be guaranteed
there — describing the retired scheme is the rubric's job, not the template's.

**`references/audit-checklist.md`** — R1 and R3 rewritten from scratch; R6 and R6b lose their fold
requirements.

- **R1** now requires the digest `@`-reference and the folder read policy. *Outdated looks like:*
  `@docs/CHANGELOG.md` still in required reading after a migration (64k tokens for a file nothing
  writes); no digest pointer, so distant history is unreachable; the window stated without a
  bound, or omitted entirely, leaving the agent to read nothing or everything.
- **R3** carries the folder convention, the two-tier entry shape, and the surviving quality rules.
  *Outdated looks like* keeps its layout-independent finds verbatim — no state-delta/why framing,
  brevity framed as "as short as possible" with no per-line bound — and gains the layout finds:
  a fragment-and-fold section (retired staging area, and it now points at a directory the
  convention no longer uses); a numbering or archive rule; a lead with no bound while the detail
  body runs unbounded.
- **R3's migration rubric** classifies a file still on the single-file scheme. **OK, not a
  finding** when no archive file exists, no duplicate numbers exist, `docs/CHANGELOG.md` is under
  ~400 lines, and R6 is direct-to-`main` with R6b absent — mention the folder convention once in
  the run summary as available on request. **Outdated** on any one of: an archive file exists (an
  archival pass already happened, so the toil is recurring); duplicate numbers exist
  (`grep -hoE '^[0-9]+\.' docs/CHANGELOG.md docs/CHANGELOG-archive.md 2>/dev/null | sort | uniq -d`,
  across both files, since a cross-file duplicate is the same defect); the file is past ~400 lines;
  or the repo commits via Branch+PR (R6), or worktrees are in use **per the audit's a2 answer**
  (R6b) — worded as the answer rather than as the subsection's presence, since a file with no
  worktree subsection whose owner answers *yes* is textually absent and semantically yes, and two
  auditors would otherwise classify it differently. Parallel landings make the numbering invariant
  unenforceable regardless of size.
- **A repo whose layout audits OK can still have an orthogonal R3 finding** — brevity framed as
  "as short as possible", or no state-delta/why framing. Fixing those must not leak the folder
  convention into a repo the rubric has just said should keep its single file: the fix is applied
  *inside the file's existing convention*, preserving its numbering, append-to-the-end and archive
  wording verbatim. R3 has to say this explicitly, because the only layout any other skill file
  describes is the folder one, and the generic "rewrite to the current recommendation's intent" in
  *Applying updates* would otherwise pull the whole layout in behind a brevity fix.
- **The backfill and the distillation are inside the atomic item, not offered alongside it.** A
  migration that converts the layout but leaves the folder and the digest empty has taken 64k of
  history out of required reading and given nothing back — and the repos that trigger the migration
  are exactly the ones with the most to lose. Both run as one pass over the original changelog,
  because both need the same read: split every entry into a dated file with its number recorded
  (*The backfill* above), and lift the digest out of what that pass sees — the spine is the
  milestone summaries the archival passes already left behind (flux has six), then a sweep for the
  class of entry that makes distant history load-bearing at all: a decision with a rejected
  alternative, something tried and removed, a constraint that still binds. The digest's ~100-line
  ceiling forces aggressive selection; 537 entries do not become 537 digest lines, even though they
  do become 537 files. **Delegate it as a fan-out over entry ranges** — ~470 KB of reading plus
  hundreds of writes is not an inline pass in the audit session. **State the cost at the offer**,
  and that the dates it produces are synthetic. A user who won't pay it declines the migration and
  keeps the single file, which is a supported state — the triggers are evidence of pressure, not a
  deprecation notice.
- **The migration is one item across R1, R3 and R6/R6b** — the existing R3+R6/R6b coupling note
  is the precedent, and the failure mode is sharper here: R3 alone routes entries into
  `docs/changelog/` while R1 still requires reading a file nothing writes, so new entries become
  invisible at session start; R6/R6b left alone still runs a fold against a directory that no longer
  exists; and the layout without the distillation is the memory loss described above. Offer all of
  it or none of it.
- **R6** loses the "Branch+PR with no fold rule" bullet; a mode still describing a fold is the R3
  migration finding, cross-referenced, not its own. Its direct-to-`main` "when complete" bullet is
  untouched.
- **R6b** loses the fold bullet (line 76) entirely, including the `--no-commit` requirement and
  the residual-renumber requirement. Its detection rule, `--path-format=absolute` check, landing
  mechanics, review-gate placement, keep-or-remove question and fan-out distinction are all
  untouched.

**`SKILL.md`** — description line 3 (`seeds an empty docs/CHANGELOG.md` → seeds the digest; the
parenthetical example of an outdated changelog section); intro line 8; step 4's fragment-bullet
paragraph at 65 **deleted** along with its two trim clauses, since the rule is now
mode-independent — step 4's question (b) survives untouched, it just stops having a changelog
consequence; step 6b's example finding at 118; step 6's narrowing exception at 129 retargeted to
the layout migration; step 7 (133–148) rewritten to seed `docs/CHANGELOG-DIGEST.md` — guarded so it does **not** seed a
digest into a repo whose audit just classified the single-file scheme OK, which would leave a digest
beside a changelog whose `CLAUDE.md` never mentions it; step 10
gains one bullet noting the window read is an instruction the session performs rather than an
`@`-reference; safety line 217 gains the append-only carve-out; scope line 221 swaps the seeded
file and adds the migration.

**`README.md`** — lines 3 and 5 both describe the scheme ("seeds `docs/CHANGELOG.md`", "numbered
append-only changelog"). Not in the original anchor list, but leaving them makes the repo's own
front page describe the retired scheme.

## Alternatives considered

- **Numeric prefix + rename-on-landing** (`0535-slug.md`). Keeps `entry N` addressing for new
  entries, at the cost of retaining a landing step, needing a janitor rule for duplicate
  prefixes, and keeping the *silent* failure: a forgotten `git mv` produces a clean-merging
  duplicate, exactly the collision the fragment rule blocked outright. It also re-breaks what the
  fragment rule bought — the reviewed artifact and the landed artifact stop being the same file.
  Date-slug inverts the failure: two entries claiming one path is an add/add conflict, which is
  blocked, not invisible.
- **Keeping the single file for repos under ~50 entries.** Costs the skill two rubrics, two
  template variants and a threshold judgement in every audit, and contradicts the no-file-
  describes-both-schemes constraint. The deciding asymmetry: the folder degrades gracefully at
  small N, the single file does not at large N. Concurrency, not size, is what breaks numbering —
  flux's duplicates were written by same-day sessions.
- **A generated `RECENT.md` that can be `@`-referenced.** Restores a single loadable file, but a
  generated index is a fold step by another name and goes stale silently. Rejected on the same
  grounds as the fold.
- **Sharding the folder by year.** Adds a rule and a path-construction decision to every write,
  and breaks the one-command index. `ls | tail` over 537 files needs no help.

## Implementation strategy

*Not part of the design — a starting point for whoever builds this.*

- **Single agent, Opus 5.** Four files whose sections cross-reference each other: the template must
  describe only the folder scheme while the rubric must describe the retired one in order to detect
  it, and `SKILL.md`'s steps 4/6/7 have to agree with both. That boundary is one continuous
  judgement, not a set of independent edits — a fan-out would split the two halves of it across
  agents that can't see each other's wording.
- **The completion check is mechanical and should be run last:** `grep -rn 'changelog\.d\|fragment\|
  fold\|CHANGELOG-archive\|monotonic' SKILL.md README.md templates/ references/` — every surviving
  hit must be inside a rubric passage that describes the retired scheme *as retired*.
