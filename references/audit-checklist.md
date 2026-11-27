# Audit checklist for existing CLAUDE.md files

This is the rubric the skill uses when a `CLAUDE.md` already exists. The job is **not** to diff against the raw template — an existing file will have been customized, reordered, and grown project-specific sections. The job is to judge whether each *recommendation below* is satisfied, and to surface only the gaps.

A recommendation is satisfied when its **intent** is met, even if the wording differs from the template. Don't flag a section just because it's phrased differently — flag it only when the intent is missing, weakened, or stale.

**Length is part of the intent.** The generated file loads into every session and is meant to state rules, not argue for them. A template-owned section that still carries an earlier version's reasoning — paragraphs explaining or defending each rule, cost estimates, objections answered in advance — is **Outdated** even when every rule in it is right. Cut it to the template's rules; any rule the project added stays.

**So is tone.** Forceful framing — "these rules are not optional", "you MUST", capitals for emphasis — is **Outdated** wherever it appears: current models follow a plain rule, and pressure makes them over-apply it. Restate the rule plainly. In a section the project wrote, that's a minor finding the user may decline; the rule itself stays.

## How to classify each item

For every rubric item below, classify the existing file as one of:

- **OK** — intent is present and current. Not a finding; don't mention it.
- **Missing** — the recommendation is absent entirely.
- **Outdated** — present, but reflects an older convention than the current recommendation (see the "Outdated looks like" notes — these are the high-value finds). A section the template has **retired** is Outdated too.
- **Stale** — present, but references something that no longer exists (a deleted subtree, a renamed doc, a leftover `<placeholder>`).

Each finding gets one line: the file, the item, the classification, and a few words of *why it matters* so the user can decide. Keep it scannable.

A leftover unfilled placeholder (`<like this>`) anywhere in an existing file is always a **Stale** finding — it means a previous scaffold run was never completed.

**History-only wording is Outdated even where the rule's intent is met.** This means a sentence that defines a current rule by contrast with a mechanism the repo doesn't have — "there is no summary file to load in its place", "now that entries no longer share a file", "there is no end-of-turn nudge". It's a finding because a session that never saw the older version reads the sentence literally, as a pointer to an alternative, and goes looking for it. The fix restates the rule as the only rule there has ever been (*Writing CLAUDE.md text* in `SKILL.md`). A sentence that guards against something a fresh session would do on its own is not this finding: "there is no `CHANGELOG.md`" stops a session from creating one, so it stays.

---

## Root `CLAUDE.md` rubric

Source of truth: `templates/CLAUDE-root.md`.

### R1 — The docs, loaded with `@`
`@docs/CONCEPT.md` and `@docs/ARCHITECTURE.md`, each with a short label — what the project does, how it's built — under a *Source of truth* heading. The label names the doc's role; it doesn't summarize the doc, which loads right below it.
- *Why:* anchors every session in the contract docs. The `@` loads them at session start, so no separate instruction to read them is needed.
- *Outdated looks like:*
  - **an "Always read first" block carrying the old changelog reading ritual** — read the newest 10 entries or the last seven days, delegate a digest of the rest to a subagent, read `docs/CLAUDE_TODO.md`. Cut it to the two `@` lines; the changelog's read is R3's `ls` plus grep.
  - **`@docs/CHANGELOG.md`, `@docs/CHANGELOG-DIGEST.md`, or any curated digest or decision log in required reading** — drop the reference, and report the file under R3's old-layout check.
  - **a project description under the title that repeats the opening of `CONCEPT.md`** — it loads twice and drifts from the doc. Drop it.
  - plain links instead of `@`-references.

### R2 — Source-of-truth statement
States that CONCEPT/ARCHITECTURE are the source of truth for *what* and *how*; that they run ahead of the code, so something not built yet is the work rather than a mismatch; and that where existing code does something differently, the agent changes neither side to match the other and asks which is right.
- *Why:* forces doc/code reconciliation instead of silent drift in either direction. The "ahead of the code" sentence stops the agent reading every unbuilt part as a contradiction, or assuming something exists because a doc describes it.
- *Outdated looks like:* absent, or softened so the agent isn't told to ask; **"flag it rather than guessing"** with no action — it doesn't say whether to stop, carry on, or fix one side — so rewrite it to the template's wording.
- *Not a finding:* the "run ahead of the code" sentence missing. The owner drops it once the code has caught up.

### R3 — Changelog
Every change recorded in `docs/changelog/` as **its own file**, `YYYY-MM-DD-<slug>.md`, written in the same change; no `CHANGELOG.md`, no entry numbers, no digest file. Also present: `ls docs/changelog` and a grep over the folder **before touching an area**; the slug names the thing, not the activity; a **lead** of 1–5 short lines saying what is now true that wasn't before, and why when that isn't obvious — not a diff recap; a **`## Detail`** for rejected alternatives, constraints that still bind, known gaps and what verification didn't cover, in words someone would grep for, never restating the lead, left out when there's nothing; entries never renamed, moved or deleted, and a change revisiting an earlier decision cites that entry's slug; the bad/good example.
- *Why:* durable project memory reached by the index and a grep, with no shared file for parallel changes to collide on. The example steers entry quality better than any description of it.
- *Outdated looks like:*
  - **the earlier long form** — lead bounds argued at length, a ~40-line detail ceiling, *Where `HEAD` is doesn't matter*, the worktree base-copy bullet, the "deferred follow-up gets written twice" bullet pointing at `docs/CLAUDE_TODO.md`, the findability essay. Cut to the template's bullets.
  - **frames the content as "summarize what changed"** with no notion of state-delta / why / not-from-the-diff — produces low-value diff recaps.
  - **frames brevity only as "one line" / "as short as possible"** with no per-line bound — gameable, since the model then packs everything onto one long line.
  - **no immutability rule** for landed entry files, so a tidy-up rename breaks every citation with nothing to catch it.
  - **no grep step**, leaving the agent holding a list of names it was never told to open.
  - **a digest rule** — milestones also earning lines in `docs/CHANGELOG-DIGEST.md`. Remove the rule, and report the file under the old-layout check.
  - **a fragment-and-fold rule** — entries staged in `docs/changelog.d/` and folded into one file at landing. Remove it on a repo already on the folder.
- **Old layout: report it, don't convert it.** Run `test -f` on `docs/CHANGELOG.md`, `docs/CHANGELOG-archive.md` and `docs/CHANGELOG-DIGEST.md` once per audit. A hit is history the per-entry folder doesn't hold: report it as one finding naming the files, and offer no fix — the skill no longer migrates them. A file whose changelog section still describes the single-file numbered scheme is left untouched, since rewriting it to the folder convention would leave the existing history unread. Never edit, move or delete those files.

### R4 — Retired: nested guidance pointers
Earlier versions listed each subtree `CLAUDE.md` with a one-line scope. Claude Code loads a subtree's `CLAUDE.md` when it reads files there, so the list only duplicates that, and goes stale when a subtree moves.
- *Outdated looks like:* a *Nested guidance* section.
- *Fix:* remove it. A rule the project put inside it — e.g. "a change to a shared type must reach both backend and frontend" — moves to the subtree `CLAUDE.md` it belongs in, unless it's already there. List each moved or dropped sentence in the finding.

### R5 — Review at commit time (one review point)
One line under *Commits*: when the user asks to commit a non-trivial diff, the agent first asks with `AskUserQuestion` — run `/code-review high --fix` (recommended), or commit now — and skips the question for a trivial diff. The agent runs the pass; the user only chooses whether it happens.
- *Why:* one review of the accumulated diff, at the moment it's sealed into history, is where a review has the most to see and the least to re-litigate. A pass per prompt is mostly waste: it only ever sees its own turn's edits, reviews them from inside the context that just wrote them, and the next follow-up re-opens what it looked at.
- *Project-specific additions are the project's call* — a risk-surface list for the agent to call out at the gate is fine. Only the structural drifts below are findings.
- *Outdated looks like* (a per-edit mandate is the highest-value find in any file scaffolded before this convention):
  - **the earlier long form** — a *Reviewing changes* section with the reasoning for the commit as review point, the once-per-commit and delegated-commit rules, and a *Running the review pass* subsection. Cut to the line, keeping any project-specific risk list;
  - **a per-edit review mandate** — "after a non-trivial edit, the change gets a review automatically, in the same turn", with the commit gate as an extra on top or absent. Replace the per-edit trigger with the gate;
  - **no commit gate** — nothing stops a commit no review has seen, so the accumulated diff is never read as one change;
  - **a review path other than `/code-review high --fix`** — `/fix-code --fix` (alone, as a preference, or as a second option), a hand-written inline pass, or a gate itself *gated on* `fix-code` being installed. Collapse it to the template's line. The explicit `high` stays: without a level, `/code-review` reuses whatever level was typed last;
  - tells the agent to **ask the user which findings to fix** instead of auto-fixing all of them — e.g. `/code-review high` without `--fix`;
  - **routes the user to a separate review pass** — an end-of-turn nudge to run some review command. The commit gate is the one sanctioned stop-and-ask and must **not** be flagged as this drift: it fires only on an explicit commit request, and the agent runs the pass itself;
  - **defers verification along with the review** — the file says tests / build / browser checks also wait for the commit. Only the review does.

### R6 — Git: where commits land
One bullet under *Commits*, for one of **two** modes: commit straight to `main` without a branch (plus "push after each commit" only if the user wants it), or branch from `main` per change under the naming convention and open a PR (who merges, whether `main` is protected).
- *Why:* the agent's generic default is to branch first when on `main`. A repo that commits to `main` has to say so, and a PR repo has to name its branches.
- *Missing looks like:* no git guidance at all, so the agent falls back to its default. Filling it in **requires asking the user** — the choice can't be inferred from the repo.
- *Outdated looks like:*
  - **the earlier long form** — a *Git workflow* section with *Where commits land*, the "where, not when" paragraph, the changelog-needs-nothing-at-landing paragraph, and the review-gate pointer. Cut to the bullet. "Commit only when asked" is Claude Code's own default and goes too;
  - **a line telling the agent to commit when a change is complete** — remove it; committing is user-initiated by default;
  - **Branch+PR describing a changelog fold or fragment at landing** — remove it on a repo already on the folder (R3).

### R6b — Retired: worktree guidance
Earlier versions of this skill wrote a *Working in a worktree* subsection — detection, branch naming, the landing sequence, a keep-or-remove question, and what a fresh worktree needs — and asked whether worktrees were used here. Worktree mechanics now belong to the skill that creates the worktree: a builder creates, sets up, lands and removes its own. A second copy in `CLAUDE.md` contradicts it and drifts (one repo already named its branches `worktree-<slug>` while the template said `wt/<slug>`).
- *Outdated looks like:* any *Working in a worktree* subsection, or worktree landing rules in any form — including the old three-mode wording ("Direct to `main`, parallel work in worktrees").
- *Fix:* remove the subsection, and then:
  - move any rule about what reaches `origin` — "never push any branch but `main`" — into the *Commits* section, if it lives only in the subsection;
  - drop everything else, including the base-copy sentence, fresh-worktree setup lists and safety rules about other sessions' servers or uncommitted work — the builder owns those. This is one of the places the audit drops sentences a project wrote, because they sit inside a section the template used to own, so **list each dropped sentence in the finding**, and the diff and confirm show them before anything is written;
  - leave `.gitignore` and `.git/info/exclude` entries for `.claude/worktrees/` alone — builders still create worktrees there.
- *No question:* the file shows everything this item needs, so the audit never asks about worktrees.
- *Absent is OK:* a file with no worktree guidance has no finding here.

### R7 — Retired: multi-agent cost-tiering
Earlier versions carried a *Multi-agent workflows* section tiering subagent models for the Workflow tool ("ultracode"). Nothing in it is about the repo, so it doesn't belong in a file every session loads.
- *Outdated looks like:* the section present.
- *Fix:* remove it. Say in the finding that it belongs in the user's own `~/.claude/CLAUDE.md` if they want it.

### R8 — Retired: owed follow-ups (`docs/CLAUDE_TODO.md`)
Earlier versions carried an *Owed follow-ups* section, a read of `docs/CLAUDE_TODO.md` in the read-first block, and a changelog bullet writing each deferred follow-up to both places.
- *Outdated looks like:* any of the three.
- *Fix:* remove them. **Leave `docs/CLAUDE_TODO.md` itself exactly as it is** — never create, edit or delete it. Say in the finding that nothing points at the file any more, so the user decides what happens to it.

### R9 — Retired: `TODO.md` is the user's
Earlier versions carried a section saying the agent never writes to `TODO.md`.
- *Outdated looks like:* the section present.
- *Fix:* remove it, and touch no `TODO.md`. A rule about `TODO.md` the project wrote elsewhere is the project's and stays.

---

## Subtree `CLAUDE.md` rubric

Source of truth: `templates/CLAUDE-subtree.md`. Apply per subtree file. Every line in the template's sections is judged by one test: would a session that has read `docs/ARCHITECTURE.md`, the manifest and the folder act differently without it? A line that fails is Outdated, and the fix removes it.

### S1 — Retired: scope line
Earlier versions opened each subtree file with a sentence on what lives there, sometimes a contents list. By the time the file loads, the session is reading the folder, and `ARCHITECTURE.md` describes it; a list also goes stale and names folders nobody has built.
- *Outdated looks like:* a description sentence or a contents list under the title.
- *Fix:* remove it. An entry point the folder doesn't make obvious moves into the rule that needs it, if one does.

### S2 — Retired: required tools and skills
Earlier versions listed the tools and skills a subtree needs (`LSP`, Playwright MCP, a design-system skill, `security-review`, `frontend-design`, `claude-api`). A session already has every tool installed on the machine, and a skill runs when its own description matches the work. A mandate repeats that, makes a skill run where it doesn't fit, and fails for anyone who hasn't installed it.
- *Outdated looks like:* a *Required tools* or *Required skills* section.
- *Fix:* remove it. A tool a step uses stays in that step, as Playwright MCP does in the verification workflow (S4). A line that is really a rule about the subtree's code ("check an option in the SDK's installed type definitions before writing a call to it") moves to the subtree's rules. In the finding, say that a general tool preference (use `LSP` instead of grep) belongs in the user's own `~/.claude/CLAUDE.md` if they want it, and that a project skill which doesn't trigger reliably needs a better description in the skill itself.

### S3 — Testing
Four kinds of line only: what tests must never touch and the fake that stands in for it; a check the test script doesn't run, or one that must run from elsewhere to catch breakage outside the subtree; a script to use where the bare tool skips a pre-step; a coverage rule the tests already follow ("every endpoint has a route test").
- *Why:* a session reads the framework, the test command and where tests live from the manifest and the folder. What it can't see is what makes a test run dangerous, or a check incomplete.
- *Outdated looks like:* **lines a session sees by itself** — the framework's name, "tests sit next to the code", "run `npm test` before you report a change complete", "do not introduce a different test framework". Remove them; a check the test script doesn't run (a type check) survives as its own line. **A bare tool call** (`tsc --noEmit`, `pytest`) where the manifest has a script for it — the script carries pre-steps such as a generated file, and the bare call skips them.
- *Missing looks like:* **a real resource with no never-touch line.** Go through each real resource the subtree's code uses — an external API, a repo, a database, a mailer, the user's data folder — and check that a line names it and the fake or temp copy tests use instead. A line about one resource doesn't cover another. Fill it from the helpers and the tests.

### S4 — Verification workflow (UI subtrees only)
Opens with "For a change to what the page shows or does, before you report it complete:", then: start the app with the exact command and the setting that points it at scratch data → drive the changed feature via Playwright MCP → check console + network for errors.
- *Why:* a passing test suite doesn't prove a UI change works, and a check on the user's real data can damage it.
- *Missing looks like:* a UI subtree with no browser-driven verification step. (Backend/library subtrees don't need this — don't flag its absence there.)
- *Outdated looks like:* a start step on the user's real data when the app can be pointed at scratch data (an env var for the database path, a seed command); **"for every UI change"**, which read literally costs a browser run for a typo; a claim about the machine ("headless", "installed") that is false on another one.

### S5 — Retired: merged into S2

### S6 — Only what `ARCHITECTURE.md` and the tests don't already cover
Rules about the subtree's code that the doc doesn't state and no test enforces.
- *Why:* every session loads `ARCHITECTURE.md`, so a rule copied from it is a second copy, and one of the two drifts. A rule a test enforces fails the test run whether or not the session read it.
- *Outdated looks like:* **a rule that only restates `ARCHITECTURE.md`**, often marked "from `docs/ARCHITECTURE.md`", including one about a file the doc already calls generated; **a rule a test or lint script already enforces** ("the lint config rejects inline styles"). Remove it — unless it adds an action or a location the doc doesn't give ("retries are decided in `queue/retry.ts`"), or a code comment cites it: grep the code for the subtree file's path before removing anything, and keep what a comment points at. List each removed rule in the finding.
- *Suggestion, not a fix:* **a mechanical self-check written as steps** ("search your diff for `style=` and hex colors"). A test would enforce it for every session. Name it in the finding and in step 10's summary; the audit edits `CLAUDE.md` files only, so it doesn't write the test.

### S7 — A file with nothing left
- *Outdated looks like:* a subtree file in which no line survives once the other findings are applied, only its title.
- *Fix:* delete the file, shown in the diff like any other change, and say in the finding that the subtree has no rules of its own.

---

## Applying updates (surgical)

When the user picks a file to update, apply only its findings, and preserve everything else:

- **Missing** → add the section, filled from the project the same way a fresh scaffold would (read the manifest / architecture doc for the real values; don't paste raw placeholders).
- **Outdated** → rewrite that section to the current recommendation's intent and the template's length, but keep any project-specific wording and rules wrapped around it. Don't reword satisfied parts just to match the template. A **retired** section (R4, R6b, R7, R8, R9, S1, S2) is removed: a project-specific rule inside it moves to where it now belongs, and every sentence the project wrote that gets dropped is listed in the finding.
- **Stale** → fix it (update the pointer, fill the placeholder) or, for a deleted-subtree pointer, remove that one line.
- **Write the current rule, not its history.** This rubric names retired sections and schemes so the audit can spot them. That history belongs in the finding shown to the user, never in the text written to the file (*Writing CLAUDE.md text* in `SKILL.md`): a file whose *Owed follow-ups* section was removed simply doesn't have one, with no sentence saying it went. A replacement for a cut-down section reads as though the section had always said it.
- **Never delete** custom sections the template never had — forcing rules, naming conventions, deployment notes. They're the whole reason the update is surgical and not a regeneration.
- Always **show the diff and confirm** before writing. A file the user didn't pick is left exactly as-is.
