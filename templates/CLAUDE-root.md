# <Project name>

<One-sentence description of what the project does.>

## Always read first

Before doing any work in this repo, **always read all** of these:

- @docs/CONCEPT.md — <one line specific to this project: what the system does, primary domain entities, key lifecycles, UI/UX scope>
- @docs/ARCHITECTURE.md — <one line specific to this project: stack choices, topology, state model, project layout>
- @docs/CHANGELOG.md — running log of changes to this project.

`CONCEPT.md` and `ARCHITECTURE.md` are the source of truth for *what* we're building, what we've built, and *how* it's structured. If something in the code contradicts them, either the code or the doc is wrong — flag it rather than guessing.

## Changelog discipline

Every change you make to this repository must be recorded in `docs/CHANGELOG.md` — written straight in, or staged in a fragment first where the fragment bullet below applies.

- Each entry is numbered with a monotonically increasing integer (1, 2, 3, ...). Never reuse or reorder numbers.
- Append new entries to the end of the file.
- Write each entry as **durable project memory, not a recap of the diff**: record what is now *true that wasn't before* — new behavior, state, or rule — plus, in a clause and only when it isn't obvious, the *why*, the alternative you rejected (so a future agent doesn't re-introduce it), or a known limit / deferred follow-up. Skip filenames, mechanical edits, and refactors with no behavior change; the diff and commit already hold those. Self-check: *if a future agent reads this entry before the code, does it learn what changed, why it matters, or what's now safe to assume?* If not, it's noise.
- Keep each entry to **1–5 lines, ~20 words per line at most**. The changelog is read at session start to orient — that only works if it stays scannable. The failure mode to avoid is cramming everything onto one unbroken line: a 40-word run-on isn't a short entry, it just hides the bulk on a single line. Break it into a few short lines instead; and if it sprawls past ~5 lines, that's a signal it's really several changes — give each its own numbered entry.
- Write the entry as part of the same change. Do not batch multiple changes into one entry, and do not skip entries.
- When a phase/increment completes, its per-task entries move to `docs/CHANGELOG-archive.md`, leaving only the milestone summary in `docs/CHANGELOG.md`. Numbers are globally unique across both files — never reuse one that already appears in either.
- **Whenever `HEAD` is not `main`** — a PR branch, or a worktree's branch — **do not write to `docs/CHANGELOG.md` from there at all.** Only a session committing directly onto `main` writes the file directly, because only there is the next number certain to still be free when the commit lands. *Being in the main checkout is not the test:* under *Branch + PR* the feature branch is normally checked out right there, and a numbered entry written on it is exactly as stale as one written in a worktree. Every parallel pair collides otherwise: both reach for the same next number, and both append to the same end of the same file. The entry goes to a **fragment** instead — `docs/changelog.d/<slug>.md`, named for the branch or worktree so no two can ever touch the same file, holding the entry body **with no number**. It goes in with the change, and it is the form the entry is in when the review gate reads the diff — the number is added later and is never what gets reviewed. It is folded into `docs/CHANGELOG.md` — numbered, and the fragment deleted — as part of landing, so `docs/changelog.d/` is empty on `main` at all times and the session-start read of the changelog is never partial. That emptiness is the invariant the scheme rests on, and it is the one thing here that can break silently — fragment filenames never collide, so an unfolded one merges to `main` cleanly instead of being blocked. **If you ever find files in `docs/changelog.d/` on `main`, fold them on sight.** The fragment needs no commit of its own: where the only commit is the landing commit, it is written and folded inside that one commit and never appears in history. Nothing above changes: the number is simply claimed at the fold, when `main` has just been merged in and `max + 1` is current by construction. Mechanics in *Git workflow* below.

Same change, bad vs. good entry:

- **Bad** (short, but just recaps the diff — zero orientation value): `42. Updated auth files, reworked middleware, added tests, renamed AuthHelper.`
- **Good** (states what's now true, with the why in a clause):
  ```
  42. Auth now rejects expired refresh tokens before session lookup; stale sessions can no longer silently renew.
      Validated at the middleware boundary so handlers can assume requests are current.
  ```

## Nested guidance

Each `src/` subtree (or service / package / area) has its own `CLAUDE.md` with scoped tool/skill rules:

- `src/<subtree-a>/CLAUDE.md` — <one-line scope>
- `src/<subtree-b>/CLAUDE.md` — <one-line scope>

## Reviewing changes

**This repo has exactly one review point: the commit.** Do not review after every edit, and do not run an unrequested review pass mid-session. A pass per prompt costs more than it catches — it only ever sees its own turn's edits, and the next follow-up re-opens what it just looked at.

What does *not* wait for the commit is verifying your own work: run the tests, the build, and whatever the subtree's `CLAUDE.md` mandates, per change, as always. Deferred here is the *review*, not the checking. Never report a change complete on the grounds that its review comes later.

The commit is the right moment because it's when the diff is sealed into history, and because it's the first point where the accumulated diff can be read as **one change** — which is what a review needs. Reviewed edit-by-edit, from inside the context that just wrote each one, a review sees the least and repeats its own blind spots.

**So when the user asks you to commit (or commit and push) and the working tree holds non-trivial changes, ask before committing** — `AskUserQuestion`, three options in this order:

1. **`/code-review high --fix`** — the deep pass: broad coverage, correctness plus reuse / simplification / efficiency, repairs applied to the working tree.
2. **`/fix-code --fix`** — the tight pass: every finding rated by consequence and independently verified, only the serious and unambiguous ones repaired.
3. **Commit now** — skip the review and go straight to the commit.

The first two are both real reviews; they differ in breadth and cost, not in whether the change gets looked at. **Option 1 leads because it catches the most** — recommend it for the diff this gate usually sees (accumulated over several rounds, or landed by a fan-out), and option 2 when the diff is small, contained, and already well understood.

**Label each option with the path that will actually run**, so the choice is concrete rather than abstract. `/code-review` ships with the harness, so option 1 is always available exactly as written — keep the explicit `high`, because without a level the command silently reuses whatever level was typed last. `/fix-code` is an installed skill: when it isn't present, option 2 becomes **"Review inline"** and you run that review yourself. Don't hide which of these the user is about to get behind a generic label — they differ in cost and in calibration.

**You** run the pass; the user is only choosing whether it happens and how deep it goes. This is the one place the review flow stops to ask — there is no end-of-turn nudge to go run a review elsewhere.

- Ask **once per commit request**, not per round or per file. If the user picks a review, run it, report, then continue to the commit without asking again.
- **Skip the question** when the diff is trivial — a typo, a version bump, a changelog line — or when a full review pass has already covered this working diff since the last edit. Asking there is noise.
- **Any** commit request goes through the gate, including a delegated one (`/git commit`, `/git commitandpush`) — check it before handing off, not after. A skill or subagent that does the committing never sees this rule.
- If the user picks *Commit now*, that's the answer — commit as asked, and don't re-offer or hedge about it afterwards.

### Running the review pass

**Option 1 — `/code-review high --fix`.** It reviews the current diff at high effort: broader coverage than the lower levels, including findings it isn't fully certain of, which is what you want at a gate that fires once per commit. Its brief is wider than option 2's — correctness bugs *and* reuse / simplification / efficiency cleanups in the same pass — and `--fix` applies what it finds to the working tree after the review. Its own report stands; don't restate it. Being the broader path, it will also repair things option 2 would have deliberately left alone, so read the resulting diff before committing rather than assuming every edit was a blocker.

**Option 2 — `/fix-code --fix`, whenever that skill is installed.** It resolves the diff scope itself, rates every finding 1–5 by consequence, has an independent verifier refute each one before repairing, takes a restore point first, and applies the repairs that are both serious and unambiguous — reversible via `/fix-code --undo`. Its own report stands; don't restate it. Two things it deliberately does *not* do: it leaves severity-1 and -2 findings unrepaired, and it treats style / naming / reuse as `/simplify`'s job and deep security work as `/security-review`'s. Run those separately if the change warrants them.

**When `fix-code` isn't installed, option 2 is that same review run by you, inline over the working diff.** A missing skill doesn't remove the review — it only changes who performs it. Scale it to the diff:

- **Small, contained diff** — read it yourself in one careful pass.
- **Substantial or complex diff** (what an accumulated multi-round working tree usually is) — fan the review out across **subagents via the Agent tool** (individual agents; this needs no ultracode opt-in — that gate is only for the Workflow tool), **one per dimension that's actually at risk in this diff (typically 2–4)**, then **verify each finding before acting on it**. Review finders run on the strong model; verification can drop a tier (the same tiering as *Multi-agent workflows* below). If the Agent tool isn't available, do the same review yourself in one thorough pass.

Check, across the diff: **correctness, security & data-integrity, edge cases & tests, reuse / duplication, clarity, performance, and conformance to this repo's own conventions** (the CLAUDE.md rules and established patterns). Then **apply every fix to the working tree automatically** and report:

1. **Group the applied fixes by severity** — blockers (correctness bugs, data loss, security), should-fix (clear improvements, missed reuse), nits (style, naming, minor clarity).
2. **Summarize each bucket in one line** so the user can see what was fixed without expanding every finding.
3. Do not stop to ask which to fix — all findings are fixed by default. The user can read the diff and revert anything they disagree with.

Every path closes the same way: say plainly what the review couldn't settle — you guessed at intent, left a known gap, or nothing covers it — so the user knows where their own judgement is still needed. State it as a fact about the change, not as a recommendation to run anything.

## Multi-agent workflows

When you fan a task out across subagents — the Workflow tool ("ultracode") — tier each agent's model and reasoning effort to the work, so cost tracks value instead of every agent defaulting to the strongest (most expensive) model:

- **Strongest model** (the session model) — contracts, correctness-critical implementation, adversarial review (finding unknown problems), and any verification or synthesis that needs design judgment or where a wrong call silently drops a real defect (security, data-integrity, correctness blockers). Never downgrade these; they are where quality is won or lost.
- **Mid model** — build/test runners, straightforward mechanical implementation, verifying concrete already-stated findings (the review did the catching), and applying already-decided fixes.
- **Cheapest model + low effort** — docs/changelog, i18n, styling, and other boilerplate.

The guardrail: the stage that *catches* an unknown problem (review) stays strong; a stage that only *checks* or *applies* an already-identified one drops a tier by default — escalate a verifier back to the strong model only for subtle or security-/data-integrity-critical findings. For a small finding set, fold verification into the fix-apply agent (verify-and-fix in one pass) rather than one strong agent per finding. Set this per `agent()` call (`model` / `effort`); an agent that omits `model` inherits the session model, which is why an untiered fan-out silently runs everything on the most expensive tier.

**Invoking a named workflow is not authoring one.** The tiers above are yours to set only when *you* write the `agent()` calls. A built-in or named workflow carries no model overrides of its own, so every stage inherits the session model and a wide fan-out bills every agent at the top tier. Size the run before launching it — how many agents the fan-out implies, given the work it's spread over — rather than planning to retier afterwards; editing the `scriptPath` a run reports is a per-run patch, and resuming re-runs every stage from the edited `agent()` call onward, so the untiered agents get billed twice.

**A workflow's aggregate diff is what the review gate is for.** A fan-out edits files across several subagents — often in separate worktrees — so no single agent ever saw the whole combined change, and any review stage *inside* the workflow checked its own findings, not the landed diff. This doesn't earn an extra pass on return; the commit is still the review point. But when the gate asks, say the diff came out of a fan-out — it's where the deep pass earns its cost most clearly, so recommend option 1 there, and if the review falls to the inline path it gets the substantial-diff treatment above even when every agent's own slice looked small.

This section is inert unless you actually run a multi-agent workflow.

## Git workflow

<This section is **two independent choices**, not one. First: where commits ultimately land — keep one of the two modes under *Where commits land*. A mode is its **bolded line plus every unbolded paragraph up to the next bolded line** — Branch + PR is four paragraphs, not one — except the final two paragraphs of that subsection (*where, not when* and the review-gate pointer), which are mode-independent and always stay. Second: whether work here is ever isolated in a git worktree — keep or delete the *Working in a worktree* subsection. They compose. A repo that commits straight to `main` and also sometimes uses a worktree keeps **both**: the mode governs where work lands, the subsection governs how a session behaves while it's in a worktree. Delete this note.>

### Where commits land

**Direct to `main`** — when you commit, commit straight to `main`; don't open branches or PRs unless asked. <Push policy, e.g. push after each commit, or leave pushing to the user.>

**Branch + PR** — never commit directly to `main`. For each change: branch from `main` (`<naming, e.g. feat/<slug>>`), commit, push, then open a PR. <Who/what merges; note here if `main` is protected.>

Changelog entries follow the fragment rule in *Changelog discipline* above: on the branch they go to `docs/changelog.d/<slug>.md`, unnumbered, and `docs/CHANGELOG.md` is never edited from the branch. A PR is in flight for as long as it stays open, so a number written into it goes stale the moment any other PR merges — the fragment is what lets the entry sit through review without rotting.

**Fold as the last thing before you hand the PR off for merge** — not at the moment of merge itself, because when a human or an auto-merge rule does the merging, that moment isn't yours to act in. Sync the branch with `main`, append the fragment's entries to `docs/CHANGELOG.md` numbered past the highest number that sync brought in (across `docs/CHANGELOG-archive.md` as well, if it exists), delete the fragment with plain `rm`, commit and push. Folding any earlier just re-opens the race. If another PR merges between your fold and your own merge, re-sync and renumber your entry past the new max — never one that came from `main`. The fold can be its own final commit on the branch; the merge lands it with the change either way.

This mode is where the janitor rule in *Changelog discipline* earns its keep: an open PR is the likeliest way an unfolded fragment ever reaches `main`, since nothing blocks it on the way in.

**This setting only chooses *where* commits go — not *when* to make them.** Commit only when the user asks; finishing a change is not a cue to commit it. When you do commit, each commit is one complete change including its changelog entry — written straight into `docs/CHANGELOG.md`, or into the fragment where you're on a branch or in a worktree — never leave the tree half-committed.

A commit request first passes through the review gate in *Reviewing changes* above — check it before staging anything, wherever the commit is being made: a commit inside a worktree is still a commit.

### Working in a worktree

<Keep this subsection if worktrees are ever used here; delete it otherwise. Everything in it is **additive** to the mode above, and applies only while a session is actually inside a worktree.>

**Where you are is a fact to check, not a setting to remember.** The same repo gets worked both ways — most sessions in the main checkout, some in a worktree — so before you commit, determine which. `git rev-parse --path-format=absolute --git-dir --git-common-dir` prints two paths: identical in the main checkout, different in a linked worktree. The `--path-format=absolute` is required, not decorative — without it `--git-dir` comes back absolute while `--git-common-dir` comes back relative whenever you are in a subdirectory, so a naive comparison calls every subdirectory of the main checkout a worktree. The first line of `git worktree list` is always the main worktree (documented ordering), and its first whitespace-separated field is the path the landing commands below need; if that line is marked `(bare)` there is no main checkout to drive and the landing steps don't apply as written. In the main checkout, the mode above is the whole story and nothing here applies.

Parallel features each get their own **git worktree**, on a short-lived local branch that exists only to back it — `wt/<slug>` under *Direct to `main`*, or the mode's own convention under *Branch + PR*, since there the branch will be pushed as the PR branch. Under *Direct to `main`* that branch is never pushed, and is deleted along with its worktree once you're finished with it; under *Branch + PR* it **is** the PR branch, so it's pushed and follows the normal PR flow. But landing is not finishing: after a landing, continuing to commit on that branch is normal. Drift is a landed branch nobody is working in any more. Starting parallel work means creating a worktree — never run two sessions against the same checkout. <Where worktrees live, e.g. `../<repo>.wt/<slug>` or `.claude/worktrees/<slug>` — if inside the repo, it must be gitignored; and any local setup that doesn't carry into a fresh one — `.env`, installed dependencies, build caches — and so has to be re-run there.>

**Changelog entries go to a fragment while you're in the worktree.** Per *Changelog discipline* above, `docs/CHANGELOG.md` is read-only here — read it to orient (knowing it's the base copy, missing anything landed since), but write entries to `docs/changelog.d/<slug>.md`, unnumbered, using the same slug as the worktree. Several changes in one worktree append several entries to that one file, in order; each still goes in with its own change. The fold happens once, at landing. A session on `main` itself writes `docs/CHANGELOG.md` directly and needs no fragment, since its number is always current. Under *Branch + PR* that exempts nothing: even the main checkout is normally sitting on a feature branch there, so the fragment rule applies to it too.

**What counts as a landing.** Under *Direct to `main`*, "push" can only mean `main`, so **"commit and push" means land it** — the words sound smaller than the action, so say which you're doing. A bare "commit" does not: that's a checkpoint on `wt/<slug>`, which still passes the review gate and then stops there. Under *Branch + PR*, pushing the worktree's branch is just a push; landing is the merge, and the fold rule in that mode applies unchanged.

**Landing under *Direct to `main`*.** The commit on `wt/<slug>` is the review point; everything after it is mechanical and gets no second pass. You **cannot** update `main` from inside the worktree — both `git push . HEAD:main` and `git branch -f main HEAD` are refused while `main` is checked out elsewhere — so drive the main checkout with `git -C` instead. The session never has to move:

1. **In the worktree:** `git merge --no-commit main`, resolve any conflicts, fold the changelog fragment, **then** commit — one commit holding the merge, the resolution, and the fold. `--no-commit` is load-bearing: a *clean* merge commits itself, and under the fragment rule the clean merge is the normal case, since nothing on the branch touches `docs/CHANGELOG.md` any more. Without it git commits the merge out from under you and the fold is forced into a second commit.

   The fold is mechanical: append the entries in `docs/changelog.d/<slug>.md` to the end of `docs/CHANGELOG.md`, numbered past the highest number in `docs/CHANGELOG.md` and in `docs/CHANGELOG-archive.md` if that file exists (numbers are globally unique across both), then delete the fragment with plain `rm` — `git rm` errors on a fragment that was never checkpointed (untracked) or that has been appended to since (modified), and the deletion stages with the commit either way. Fold *after* the merge, never before: the merge is what makes that max current. Keep both inside the commit — a fold left for afterwards splits one change across two commits, and merging `main` in first is also what makes step 2 a fast-forward.
2. **Land it:** `git -C <main-checkout> merge --ff-only wt/<slug>`, then push per the policy above. **A refusal has two causes — read the error before reacting, because only one of them is about numbering:**
   - **`main` moved** (rejected as not a fast-forward) — the one collision the fragment can't prevent, since your entry is numbered by then. Re-run `git merge --no-commit main` in the worktree and you'll get an end-of-file conflict in `docs/CHANGELOG.md`. Resolve it by keeping both sides, the incoming entries first, and renumbering **yours** *past* the new max — new max + 1, never onto it. Commit, then retry the fast-forward. Do **not** re-fold: the fragment is gone and its entries are already in the file. You only ever renumber your own entry; never renumber, reorder, or drop one that came from `main`.
   - **The main checkout is dirty** (`Your local changes would be overwritten by merge`) — `main` has not moved and nothing needs renumbering. This is what a session working directly in the main checkout looks like from here. Commit or stash there, then retry.
3. **Tear down — if the worktree isn't being kept (see below), and from the main checkout:** `git worktree remove <worktree-path>`, then `git branch -d wt/<slug>` — in that order, since the branch can't be deleted while its worktree exists. Two traps: `git worktree remove` run from *inside* the worktree succeeds by deleting your own working directory out from under you, and every command after it fails; and `rm -rf` on the directory leaves the worktree registered and the branch undeletable until `git worktree prune`.

**Landing under *Branch + PR*.** There is no local fast-forward and no `git -C` dance: the worktree's branch *is* the PR branch, so landing is the mode's normal flow — sync, fold per the mode's fold rule, push, and the PR merges. Step 3's teardown still applies, with one correction that matters: **after a squash or rebase merge the branch's commits are not on `main` by ancestry**, so `git branch -d` refuses and you need `git branch -D`. Judge whether the work landed by the **PR's state**, never by `git log main..<branch>` — that comparison stays non-empty forever after a squash and will tell you the work is unlanded when it shipped.

**A landing asks two things — in one `AskUserQuestion`, not two stops.** The gate's one-ask-per-commit-request rule still holds; the second question rides along in the same prompt: the review question from *Reviewing changes*, and what happens to the worktree once the work has landed.

- **Keep working here** — nothing is torn down. The branch stays and keeps taking commits. Do **not** call `ExitWorktree`: its `keep` returns the session to the original directory, which is the opposite of staying.
- **Remove the worktree** — run step 3 (with the `branch -D` correction above if the PR was squash-merged). The session leaves the worktree and the next prompt runs in the main checkout; put that in the option label, because this relocates the session rather than just tidying up.

**Keep is the default** — removal costs rebuilding whatever local setup doesn't carry into a fresh worktree; keeping only costs disk. **Skip the question** when the user already said which they want. **Don't offer removal at all** while anything in the worktree isn't landed — uncommitted changes, untracked files, or commits that haven't landed: keep it and say why. Under *Direct to `main`* "hasn't landed" means not reachable from `main`; under *Branch + PR* it means the PR hasn't merged, since a squash merge leaves the commits permanently unreachable and an ancestry check would withhold removal forever. If the review reported something it couldn't settle, keep regardless and say so.

**If the session entered the worktree with `EnterWorktree`** — and this subsection is the project instruction that authorizes using it — note that it branches from `origin/<default-branch>` unless `worktree.baseRef` is `head`, so local unpushed `main` commits aren't in there. **Land before exiting:** `ExitWorktree` with `remove` refuses while the branch holds commits that aren't on the original branch, and the only way through that refusal is `discard_changes`, which destroys the work. Note that refusal is ancestry-based, so a **squash-merged PR still trips it even though the work shipped** — that is the one case where the refusal is wrong and `discard_changes` is nonetheless the wrong answer. Confirm from the PR that it merged, then tear the worktree down with `git worktree remove` and `git branch -D` instead of routing through `ExitWorktree` at all. Land first, then `remove` — and only when that's the answer to the question above; if the answer was to keep working, don't call it at all. Its `keep` action is for stepping out of a worktree you intend to return to in a later session.

The worktrees a multi-agent fan-out creates are **not** this: they're ephemeral, they don't commit, and their edits land in the parent tree to pass a single gate there. A fan-out running in the main checkout is also the most likely reason a landing hits the dirty-checkout refusal in step 2.

<!-- Add additional sections below as the project develops:
  - Project-specific forcing rules (e.g., "Check in with the user before making CSS / layout / UX changes")
  - Destructive-operation guidance if the agent's defaults aren't enough
  - Naming conventions, code-organization rules
-->
