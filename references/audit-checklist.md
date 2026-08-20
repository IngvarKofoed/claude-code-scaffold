# Audit checklist for existing CLAUDE.md files

This is the rubric the skill uses when a `CLAUDE.md` already exists. The job is **not** to diff against the raw template — an existing file will have been customized, reordered, and grown project-specific sections. The job is to judge whether each *recommendation below* is satisfied, and to surface only the gaps.

A recommendation is satisfied when its **intent** is met, even if the wording differs from the template. Don't flag a section just because it's phrased differently — flag it only when the intent is missing, weakened, or stale.

## How to classify each item

For every rubric item below, classify the existing file as one of:

- **OK** — intent is present and current. Not a finding; don't mention it.
- **Missing** — the recommendation is absent entirely.
- **Outdated** — present, but reflects an older convention than the current recommendation (see the "Outdated looks like" notes — these are the high-value finds).
- **Stale** — present, but references something that no longer exists (a deleted subtree, a renamed doc, a leftover `<placeholder>`).

Each finding gets one line: the file, the item, the classification, and a few words of *why it matters* so the user can decide. Keep it scannable.

A leftover unfilled placeholder (`<like this>`) anywhere in an existing file is always a **Stale** finding — it means a previous scaffold run was never completed.

---

## Root `CLAUDE.md` rubric

Source of truth: `templates/CLAUDE-root.md`.

### R1 — "Always read first" block
Mandates reading `@docs/CONCEPT.md`, `@docs/ARCHITECTURE.md`, and `@docs/CHANGELOG.md` before any work, using `@`-references so they load at session start.
- *Why:* anchors every session in the contract docs.
- *Outdated looks like:* missing the `@docs/CHANGELOG.md` pointer; plain links instead of `@`-references; worded as optional ("you may want to read") rather than a hard "always read".

### R2 — Source-of-truth statement
States that CONCEPT/ARCHITECTURE are the source of truth for *what* and *how*, and that contradictions in code should be flagged rather than guessed past.
- *Why:* forces doc/code reconciliation instead of silent drift.
- *Outdated looks like:* absent, or softened so the agent isn't told to flag contradictions.

### R3 — Changelog discipline
Every change recorded in `docs/CHANGELOG.md`; entries numbered with monotonically increasing integers, never reused or reordered; appended to the end; written as durable project memory (what is now *true that wasn't before*, plus the *why* in a clause when not obvious — not a recap of the diff); bounded to ~1–5 lines and ~20 words per line rather than one packed run-on line; written as part of the same change; and the archive rule — on phase completion, per-task entries move to `docs/CHANGELOG-archive.md`, with numbers globally unique across both files. Where the repo's git mode allows more than one change in flight (parallel worktrees, open PRs), it also states the **fragment** rule: `docs/CHANGELOG.md` is never written from a branch — the entry goes to `docs/changelog.d/<slug>.md` unnumbered, travels through review there, and is folded in, numbered, and deleted as part of landing, leaving that directory empty on `main`.
- *Why:* durable, ordered project memory the agent can rely on — which only works if entries stay scannable and carry orientation value, not diff noise.
- *Outdated looks like:* a changelog section that lacks the numbering rule, lacks "append to the end / never reorder", lacks the archive + global-uniqueness rule; **or frames the content as "summarize what changed"** with no notion of state-delta / why / not-from-the-diff (produces low-value diff recaps); **or frames brevity only as "one line" / "as short as possible"** with no per-line bound (gameable — the model packs everything onto one long line) instead of bounding it (~1–5 lines, ~20 words/line, split if longer). In a repo whose git mode allows parallel work, two finds: a changelog section that states the numbering invariant but says nothing about parallel changes — "monotonically increasing, never reused" then silently breaks, and surfaces as an end-of-file merge conflict with no stated resolution policy; **or one carrying the older renumber-at-landing convention** — write the numbered entry into `docs/CHANGELOG.md` on the branch, then renumber past `main`'s max before landing, redoing it if `main` moves again. That only mitigates a collision the fragment rule prevents outright: appending to one file's end guarantees a conflict on *every* parallel pair, the reviewed number stops matching the landed one, and the renumber loop is unbounded. Replace it with the fragment rule — and this is **one migration across R3 and R6, not two findings**. Offer them together and apply both or neither: R3 alone routes entries into a fragment that the old landing sequence neither folds nor deletes, so the fragment rides the merge onto `main` and the entry is never numbered at all. That is worse than leaving either convention intact.

### R4 — Nested guidance pointers
Lists each subtree `CLAUDE.md` with a one-line scope.
- *Why:* routes the agent to the scoped rules for whatever it's touching.
- *Stale looks like:* a pointer to a subtree that no longer exists, or a missing pointer for a subtree that now has its own `CLAUDE.md`. (Cross-check against the subtrees identified earlier in the run.)

### R5 — Review at commit time (one review point)
The repo has **exactly one review point: the commit** — no per-edit review pass, and no end-of-turn nudge to go run one. When the user asks to commit (or commit and push) and the working tree holds non-trivial changes, the agent **asks first** (`AskUserQuestion`): review the working diff and commit with the repairs, or commit as-is. The agent performs the pass; the user only chooses whether it happens. Asked once per commit request, skipped for a trivial diff or one a full pass already covered, and applied to delegated commits too (`/git commit`) — checked before handing off, since the committing skill never sees the rule. The pass has a two-path shape: **`/fix-code --fix` when that skill is installed**, otherwise the agent's own inline pass over the working diff, scaled to it — one careful read for a small diff; for a substantial or complex one (what an accumulated working tree usually is) a **fan-out across subagents (the Agent tool — no ultracode opt-in needed) one per at-risk dimension (~2–4), each finding verified before it's acted on** (finders on the strong model, verification a tier down — see R7). It checks correctness, security & data-integrity, edge cases & tests, reuse, clarity, performance, and conformance to the repo's own conventions; applies **every** fix automatically; reports grouped by severity (blocker / should-fix / nit), one line per bucket, without stopping to ask which to fix; and closes by stating plainly anything it couldn't fully verify. Also present: the explicit carve-out that **per-change verification is not deferred** — tests, build, and the subtree's own verification workflow still run per change; only the review waits for the commit.
- *Why:* one review of the accumulated diff, at the moment it's sealed into history, is where a review has the most to see and the least to re-litigate. A pass per prompt is mostly waste: it only ever sees its own turn's edits, reviews them from inside the context that just wrote them, and the next follow-up re-opens what it looked at.
- *The pass's rigor is the project's call* — different line counts or a different risk-surface list are both fine. Only the structural drifts below are findings.
- *Outdated looks like* (the first is the highest-value find in any file scaffolded before this convention):
  - **a per-edit review mandate** — "after a non-trivial edit, the change gets a review automatically, in the same turn", with the commit gate as an extra on top or absent. That was the previous convention here; the current one collapses the review into the commit gate. Flag it as Outdated, and when the user picks it, replace the per-edit trigger with the gate while keeping the pass description and any project-specific risk list the file added — the *frequency* is what changed, not the review's content;
  - **no commit gate** — nothing stops a commit no review has seen, so the accumulated diff is never read as one change. This is now the substantive gap (a file with the per-edit mandate *and* no gate has effectively no review point at all);
  - **makes an external review command the *only* path** — named with no inline fallback, so the review goes dead in any checkout where that skill isn't installed and the agent emits an apologetic note instead. A **conditional** preference is correct and must **not** be flagged: "`/fix-code --fix` when installed, otherwise review inline" is the current convention, and a file carrying only the inline pass is Outdated in the mild sense — offer the preference, don't rewrite the fallback. Note the older gate was itself *gated on* `fix-code` being installed ("without it, commit as asked") — that's this drift too: offer the inline fallback at the gate;
  - describes the pass as a **cursory single glance** with no subagent fan-out / finding-verification even for a large accumulated diff — the intent is review quality, not a token read;
  - tells the agent to **ask the user which findings to fix** instead of auto-fixing all of them;
  - **routes the user to a separate review pass** — an end-of-turn nudge to run some review command, gated on soft adjectives or hardcoded to one level. Fired on nearly every turn it is pure noise the user learns to skip. The **commit gate** is the one sanctioned stop-and-ask and must **not** be flagged as this drift: it fires only on an explicit commit request (discrete and infrequent, not per turn), and the agent runs the pass itself rather than routing the user elsewhere;
  - **defers verification along with the review** — the file reads as though tests / build / browser checks also wait for the commit. Only the review does; flag a file that dropped that carve-out.

### R6 — Git workflow
States how the agent should use version control here — one of three modes: commit directly to `main`; commit to `main` with parallel work isolated in **git worktrees** (short-lived local `wt/<slug>` branches that are never pushed, and are deleted with their worktree once the work has landed and the worktree isn't being kept); or branch and open a PR per change (with branch-naming / protected-`main` notes) — plus a note that the mode chooses *where* commits go, not *when*. Both modes that allow more than one change in flight also carry the changelog **fold** mechanics that R3's fragment rule depends on: where in the landing sequence the fragment becomes a numbered entry, and what happens when `main` moves inside the landing window.
- *Why:* the agent's generic default ("branch first, commit only when asked") may not match this repo; an explicit convention removes the guesswork on every change.
- *Missing looks like:* no git-workflow guidance at all, so the agent falls back to its default. Note: filling this in **requires asking the user** — the choice can't be inferred from the repo.
- *Outdated looks like:*
  - a "Direct to `main`" section that says to commit when a change is "complete" without stating that committing is still user-initiated — the agent reads it as license to commit proactively the moment work is done. Flag it and add the "where, not when" clarification;
  - a worktree mode whose prohibition lands on **creating branches** rather than on **pushing** them and opening PRs. A worktree is normally backed by a local branch, so "no feature branches" forbids the mechanism the mode depends on instead of the thing actually ruled out (a branch reaching `origin`, a PR flow). Reword to "never push any branch but `main`" and keep the delete-at-landing rule;
  - a worktree mode that doesn't say where the review gate fires. It fires at the **worktree's commit** — with `main` merged in and conflicts resolved *before* that commit, so the resolution is inside the reviewed diff — and the merge into `main` afterwards is mechanical and gets no second pass. A file that leaves this open either loses the gate entirely (the commit happens in the worktree, where nothing invoked it) or pays for two passes over one change;
  - a worktree mode that states the policy but not the **landing mechanics**, leaving the agent to discover them by hitting refusals: `main` can't be updated from inside a worktree (`git push . HEAD:main` and `git branch -f main` are both rejected while `main` is checked out elsewhere — drive the main checkout with `git -C` instead), a branch can't be deleted while its worktree exists, `rm -rf` on the worktree leaves it registered until `git worktree prune`, and `git worktree remove` run from inside the worktree deletes the agent's own working directory. Where the session uses `EnterWorktree`, add that landing must precede `ExitWorktree` — its `remove` refuses on unlanded commits, and the escape hatch (`discard_changes`) destroys the work;
  - a worktree or Branch+PR mode that **doesn't say where the changelog fold happens**. R3 sends the entry to a fragment; this section is what brings it back, and a mode that omits it leaves the entry stranded on the branch. In a worktree the fold sits inside landing step 1 — after the merge, which is what makes the target number current, and before the commit, so the change stays one commit. That requires **`git merge --no-commit main`**: under the fragment rule the branch no longer touches `docs/CHANGELOG.md`, so the merge is normally clean, and a clean merge commits itself and pushes the fold into a second commit. A landing step that says plain `git merge main` followed by "then commit" is Outdated for this reason alone — it only ever worked because the old convention guaranteed a conflict there. In Branch+PR it is the last thing before the PR is handed off for merge, after the final sync — folding earlier re-opens the race an open PR is most exposed to, but "at the moment of merge" is not a moment the agent controls when a human or an auto-merge rule merges. Also require the **janitor rule** there: per-branch fragment filenames never conflict, so an unfolded fragment merges to `main` silently rather than being blocked, and this is the only path that can break the empty-on-`main` invariant — the mode must say to fold any fragment found on `main` on sight. Also flag a mode that folds but omits the **residual case**: when the fast-forward or merge is refused because `main` moved inside the landing window, you re-sync and renumber **your own** entry only — never one that arrived from `main`. A file still describing the fold as a renumber of a `CHANGELOG.md` edit made on the branch is the R3 drift above; fix both together;
  - a worktree mode that **decides the worktree's fate silently** — tearing it down, or leaving it, without asking — or that asks in a *second* prompt after the review gate. Landing is the one moment the choice is decidable, so it belongs in the same `AskUserQuestion` as the review question, with keep as the default and removal withheld while anything in the worktree isn't landed. Also flag wording that makes *any* landed branch drift: after a fast-forward the branch equals `main` and continuing to work on it is normal — drift is a landed branch nobody is working in. And flag a mode that doesn't say **what counts as a landing** ("commit and push" can only mean `main` here, so it lands; a bare "commit" is a checkpoint);
  - a worktree mode that doesn't distinguish **feature** worktrees from the ephemeral ones a fan-out creates (R7). Fan-out worktrees don't commit; their edits land in the parent tree and pass one gate there. Without the distinction, a fan-out fragments the accumulated diff into per-slice reviews — the exact thing R5's single gate exists to prevent.

### R7 — Multi-agent workflow (ultracode) cost-tiering
When the agent fans a task across subagents (the Workflow tool / "ultracode"), it tiers model + reasoning effort per agent — strongest model for contracts / correctness-critical implementation / adversarial review; a mid model for build-test runners, mechanical implementation, verifying concrete already-stated findings, and applying decided fixes; the cheapest model + low effort for docs / i18n / styling — with the guardrail that the *catching* stage (review) stays strong while *checking* stages (verification) drop a tier unless the finding is subtle or sits in a file whose own risk is security-/data-integrity-critical (judged per file, not by classifying the whole diff — a large commit is usually mixed). It also covers **invoking a named/pre-built workflow**: those carry no model overrides of their own, so every stage inherits the session model and the fan-out has to be sized before launching rather than retiered after. And it ties back to R5: **a workflow's aggregate diff is what the commit gate is for** — no single subagent saw the whole combined change, and a review stage inside the workflow only checked its own findings, so the landed diff gets the substantial-diff treatment at the gate rather than an extra pass on return.
- *Why:* an untiered fan-out defaults every agent to the session's strongest (most expensive) model; tiering cuts cost without touching the stages that catch problems. Verification checks a finding the review already caught, so it doesn't need the strong tier by default. And a named workflow you merely *invoke* silently bills all its stages at the top tier — the trap that only-authoring guidance misses.
- *This is a suggestion, not a hard gap.* Only offer it for a repo where multi-agent workflows are plausibly used; a tiny library that will never run one doesn't need it. Absence is not a finding on its own.
- *Outdated looks like:* R7 is present but (a) covers only *authoring* (setting `model` per `agent()` call) and omits the **named/invoked-workflow** case — that invoking a pre-built workflow runs all its stages on the session model unless its script is edited to tier the checking stages down; or (b) omits the **aggregate-diff** tie-back — that a fan-out's landed diff is nobody's reviewed work and so is the clearest case for *review first* at the R5 gate. Flag either missing piece as Outdated (only where R7 is otherwise warranted); (b) is minor. A file that instead mandates a **self-review the moment a workflow returns** is carrying the older convention — same finding as the per-edit mandate in R5, fold it into that one.

---

## Subtree `CLAUDE.md` rubric

Source of truth: `templates/CLAUDE-subtree.md`. Apply per subtree file.

### S1 — Scope + contents
A one-line description of what lives in the subtree, plus a contents list.
- *Why:* orients the agent before it touches the subtree.
- *Stale looks like:* unfilled `<placeholder>` text.

### S2 — Required tools
At minimum the `LSP` for the subtree's primary language; browser tools (Playwright MCP) for UI subtrees; load instructions quoted if the tool is deferred.
- *Why:* the agent needs the right tools loaded to work effectively here.
- *Outdated looks like:* no LSP entry; a UI subtree with no browser-automation tool.

### S3 — Testing
The actual test framework named and pinned (Vitest / Jest / xUnit / pytest / …), plus the forcing rule: *do not introduce a different test framework without updating the architecture doc.*
- *Why:* stops the agent from silently fragmenting the test stack.
- *Outdated looks like:* a framework named but the forcing rule dropped; a generic unfilled placeholder where the framework should be.

### S4 — Verification workflow (UI subtrees only)
For UI subtrees, the numbered Playwright-MCP pattern: start dev server → drive the changed feature in a real browser → check console + network for errors → only then report complete.
- *Why:* a passing `tsc`/unit suite doesn't prove a UI change actually works.
- *Missing looks like:* a UI subtree with no browser-driven verification step. (Backend/library subtrees don't need this — don't flag its absence there.)

### S5 — Required skills
Project-specific skill mandates (a design system, security review, naming conventions, …).
- *Why:* chains the agent into project-specific discipline.
- *Note:* this is genuinely optional — **zero required skills is a valid choice**, so absence is **not** a finding on its own. Only flag it if there's an obvious installed, clearly-relevant skill the subtree should be mandating (e.g. a `*-design-system` skill for a frontend subtree) — and even then, frame it as a suggestion, not a gap.

---

## Applying updates (surgical / additive)

When the user picks a file to update, apply only its findings, and preserve everything else:

- **Missing** → add the section, filled from the project the same way a fresh scaffold would (read the manifest / architecture doc for the real values; don't paste raw placeholders).
- **Outdated** → rewrite that section to the current recommendation's intent, but keep any project-specific wording wrapped around it. Don't reword satisfied parts just to match the template.
- **Stale** → fix it (update the pointer, fill the placeholder) or, for a deleted-subtree pointer, remove that one line.
- **Never delete** custom sections the template never had — forcing rules, naming conventions, deployment notes. They're the whole reason the update is surgical and not a regeneration.
- Always **show the diff and confirm** before writing. A file the user didn't pick is left exactly as-is.
