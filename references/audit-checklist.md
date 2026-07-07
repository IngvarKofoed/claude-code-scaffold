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
Every change recorded in `docs/CHANGELOG.md`; entries numbered with monotonically increasing integers, never reused or reordered; appended to the end; written as durable project memory (what is now *true that wasn't before*, plus the *why* in a clause when not obvious — not a recap of the diff); bounded to ~1–5 lines and ~20 words per line rather than one packed run-on line; written as part of the same change; and the archive rule — on phase completion, per-task entries move to `docs/CHANGELOG-archive.md`, with numbers globally unique across both files.
- *Why:* durable, ordered project memory the agent can rely on — which only works if entries stay scannable and carry orientation value, not diff noise.
- *Outdated looks like:* a changelog section that lacks the numbering rule, lacks "append to the end / never reorder", lacks the archive + global-uniqueness rule; **or frames the content as "summarize what changed"** with no notion of state-delta / why / not-from-the-diff (produces low-value diff recaps); **or frames brevity only as "one line" / "as short as possible"** with no per-line bound (gameable — the model packs everything onto one long line) instead of bounding it (~1–5 lines, ~20 words/line, split if longer).

### R4 — Nested guidance pointers
Lists each subtree `CLAUDE.md` with a one-line scope.
- *Why:* routes the agent to the scoped rules for whatever it's touching.
- *Stale looks like:* a pointer to a subtree that no longer exists, or a missing pointer for a subtree that now has its own `CLAUDE.md`. (Cross-check against the subtrees identified earlier in the run.)

### R5 — "After making changes" / code-review mandate
After a non-trivial edit, explicitly invoke the **`code-review`** skill with `--fix` (default effort `medium`; `high --fix` for large or high-risk changes), apply **every** finding automatically, then report grouped by severity (blocker / should-fix / nit), one line per bucket, without stopping to ask which to fix.
- *Why:* a consistent quality gate on every change.
- *Effort level is the project's call* — do **not** flag a mandate as outdated merely because it uses `high` (a stricter, valid choice) or `medium`. Only the structural drifts below are findings.
- *Outdated looks like* (this is the most common drift, and the highest-value find):
  - invokes `code-review` with `high` but **without `--fix`**;
  - tells the agent to **ask the user which findings to fix** instead of auto-fixing all of them;
  - references an **old skill name** (e.g. a pre-rename `/review`-style name) instead of `code-review`;
  - claims the review **auto-triggers** (it does not — it must be invoked explicitly).

### R6 — Git workflow
States how the agent should use version control here: either commit directly to `main`, or branch and open a PR per change (with branch-naming / protected-`main` notes), plus a note that the mode chooses *where* commits go, not *when*.
- *Why:* the agent's generic default ("branch first, commit only when asked") may not match this repo; an explicit convention removes the guesswork on every change.
- *Missing looks like:* no git-workflow guidance at all, so the agent falls back to its default. Note: filling this in **requires asking the user** — the choice can't be inferred from the repo.
- *Outdated looks like:* a "Direct to `main`" section that says to commit when a change is "complete" without stating that committing is still user-initiated — the agent reads it as license to commit proactively the moment work is done. Flag it and add the "where, not when" clarification.

### R7 — Multi-agent workflow (ultracode) cost-tiering
When the agent fans a task across subagents (the Workflow tool / "ultracode"), it tiers model + reasoning effort per agent — strongest model for contracts / correctness-critical implementation / review / verification / synthesis; a mid model for build-test runners and mechanical implementation; the cheapest model + low effort for docs / i18n / styling — with the guardrail that only mechanical stages get a cheaper model.
- *Why:* an untiered fan-out defaults every agent to the session's strongest (most expensive) model; tiering cuts cost without touching the stages that catch problems.
- *This is a suggestion, not a hard gap.* Only offer it for a repo where multi-agent workflows are plausibly used; a tiny library that will never run one doesn't need it. Absence is not a finding on its own.

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
