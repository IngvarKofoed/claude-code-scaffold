# <Project name>

<One-sentence description of what the project does.>

## Always read first

Before doing any work in this repo, **always read all** of these:

- @docs/CONCEPT.md — <one line specific to this project: what the system does, primary domain entities, key lifecycles, UI/UX scope>
- @docs/ARCHITECTURE.md — <one line specific to this project: stack choices, topology, state model, project layout>
- @docs/CHANGELOG.md — running log of changes to this project.

`CONCEPT.md` and `ARCHITECTURE.md` are the source of truth for *what* we're building, what we've built, and *how* it's structured. If something in the code contradicts them, either the code or the doc is wrong — flag it rather than guessing.

## Changelog discipline

Every change you make to this repository must be recorded in `docs/CHANGELOG.md`.

- Each entry is numbered with a monotonically increasing integer (1, 2, 3, ...). Never reuse or reorder numbers.
- Append new entries to the end of the file.
- Write each entry as **durable project memory, not a recap of the diff**: record what is now *true that wasn't before* — new behavior, state, or rule — plus, in a clause and only when it isn't obvious, the *why*, the alternative you rejected (so a future agent doesn't re-introduce it), or a known limit / deferred follow-up. Skip filenames, mechanical edits, and refactors with no behavior change; the diff and commit already hold those. Self-check: *if a future agent reads this entry before the code, does it learn what changed, why it matters, or what's now safe to assume?* If not, it's noise.
- Keep each entry to **1–5 lines, ~20 words per line at most**. The changelog is read at session start to orient — that only works if it stays scannable. The failure mode to avoid is cramming everything onto one unbroken line: a 40-word run-on isn't a short entry, it just hides the bulk on a single line. Break it into a few short lines instead; and if it sprawls past ~5 lines, that's a signal it's really several changes — give each its own numbered entry.
- Write the entry as part of the same change. Do not batch multiple changes into one entry, and do not skip entries.
- When a phase/increment completes, its per-task entries move to `docs/CHANGELOG-archive.md`, leaving only the milestone summary in `docs/CHANGELOG.md`. Numbers are globally unique across both files — never reuse one that already appears in either.

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

## After making changes

After a non-trivial edit, **the change gets a real review automatically, in the same turn** — nobody should have to ask for it. Never skip it, and never leave it for a later pass.

**Preferred: `/fix-code --fix`, whenever that skill is installed.** It resolves the diff scope itself, rates every finding 1–5 by consequence, has an independent verifier refute each one before repairing, takes a restore point first, and applies the repairs that are both serious and unambiguous. Use it in place of the fallback below — it's better calibrated than an ad-hoc pass, and its edits are reversible via `/fix-code --undo`. Two things it deliberately does *not* do: it leaves severity-1 and -2 findings unrepaired, and it treats style / naming / reuse as `/simplify`'s job and deep security work as `/security-review`'s. Run those separately if the change warrants them.

**Fallback, when `fix-code` isn't installed: run the same review yourself, inline.** A missing skill is never a reason to skip the review — it only changes who performs it.

Scale the fallback review to the change:

- **Small, contained edits** — read the diff yourself in one careful pass.
- **Substantial or complex changes** — fan the review out across **subagents via the Agent tool** (individual agents; this needs no ultracode opt-in — that gate is only for the Workflow tool), **one per dimension that's actually at risk in this diff (typically 2–4)**, then **verify each finding before acting on it**. Review finders run on the strong model; verification can drop a tier (the same tiering as *Multi-agent workflows* below). If the Agent tool isn't available, do the same review yourself in one thorough pass.

Check, across the diff: **correctness, security & data-integrity, edge cases & tests, reuse / duplication, clarity, performance, and conformance to this repo's own conventions** (the CLAUDE.md rules and established patterns). Then **apply every fix to the working tree automatically.**

Once the fallback's fixes are applied, report what changed (when `/fix-code` ran instead, its own report stands — don't restate it):

1. **Group the applied fixes by severity** — blockers (correctness bugs, data loss, security), should-fix (clear improvements, missed reuse), nits (style, naming, minor clarity).
2. **Summarize each bucket in one line** so the user can see what was fixed without expanding every finding.
3. Do not stop to ask which to fix — all findings are fixed by default. The user can review the diff and revert anything they disagree with.

Say plainly when the change is one you couldn't fully verify — you guessed at intent, left a known gap, or nothing covers it — so the user knows where their own judgement is still needed. State it as a fact about the change, not as a recommendation to run anything.

## Multi-agent workflows

When you fan a task out across subagents — the Workflow tool ("ultracode") — tier each agent's model and reasoning effort to the work, so cost tracks value instead of every agent defaulting to the strongest (most expensive) model:

- **Strongest model** (the session model) — contracts, correctness-critical implementation, adversarial review (finding unknown problems), and any verification or synthesis that needs design judgment or where a wrong call silently drops a real defect (security, data-integrity, correctness blockers). Never downgrade these; they are where quality is won or lost.
- **Mid model** — build/test runners, straightforward mechanical implementation, verifying concrete already-stated findings (the review did the catching), and applying already-decided fixes.
- **Cheapest model + low effort** — docs/changelog, i18n, styling, and other boilerplate.

The guardrail: the stage that *catches* an unknown problem (review) stays strong; a stage that only *checks* or *applies* an already-identified one drops a tier by default — escalate a verifier back to the strong model only for subtle or security-/data-integrity-critical findings. For a small finding set, fold verification into the fix-apply agent (verify-and-fix in one pass) rather than one strong agent per finding. Set this per `agent()` call (`model` / `effort`); an agent that omits `model` inherits the session model, which is why an untiered fan-out silently runs everything on the most expensive tier.

**Invoking a named workflow is not authoring one.** The tiers above are yours to set only when *you* write the `agent()` calls. A built-in or named workflow carries no model overrides of its own, so every stage inherits the session model and a wide fan-out bills every agent at the top tier. Size the run before launching it — how many agents the fan-out implies, given the work it's spread over — rather than planning to retier afterwards; editing the `scriptPath` a run reports is a per-run patch, and resuming re-runs every stage from the edited `agent()` call onward, so the untiered agents get billed twice.

**When a workflow returns, self-review its aggregate diff.** A fan-out edits files across several subagents — often in separate worktrees — so no single agent ever saw the whole combined change. The moment control returns to you, the `## After making changes` self-review applies to the **entire diff the workflow produced**: read it as one coherent change, not per-agent, and apply fixes. A multi-agent change has no single author who saw the whole thing, so this pass matters more here than anywhere else — give it the substantial-change treatment above even when each agent's own slice looked small. (Any review stage *inside* the workflow checks its findings/outputs; it does not replace this pass over the landed diff.)

This section is inert unless you actually run a multi-agent workflow.

## Git workflow

<Keep the one mode that matches this repo; delete the other and this note.>

**Direct to `main`** — when you commit, commit straight to `main`; don't open branches or PRs unless asked. <Push policy, e.g. push after each commit, or leave pushing to the user.>

**Branch + PR** — never commit directly to `main`. For each change: branch from `main` (`<naming, e.g. feat/<slug>>`), commit, push, then open a PR. <Who/what merges; note here if `main` is protected.>

**This setting only chooses *where* commits go — not *when* to make them.** Commit only when the user asks; finishing a change is not a cue to commit it. When you do commit, each commit is one complete change including its `docs/CHANGELOG.md` entry — never leave the tree half-committed.

<!-- Add additional sections below as the project develops:
  - Project-specific forcing rules (e.g., "Check in with the user before making CSS / layout / UX changes")
  - Destructive-operation guidance if the agent's defaults aren't enough
  - Naming conventions, code-organization rules
-->
