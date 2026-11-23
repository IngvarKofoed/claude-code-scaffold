# Retire the worktree guidance

The scaffold writes a *Working in a worktree* subsection into the root
`CLAUDE.md` (`templates/CLAUDE-root.md:183-214`). It asks a dedicated question
for it (SKILL.md:63, :112), and it audits it as R6b
(`references/audit-checklist.md:133-143`). The subsection describes:

- how to tell whether the session is in a worktree;
- branch naming;
- the landing sequence;
- a keep-or-remove question at landing;
- what a fresh worktree needs.

A coming builder skill will create, set up, land and remove worktrees itself.
A second copy of those rules in every repo's `CLAUDE.md` would contradict it
and drift. It already has drifted: flux names worktree branches
`worktree-<slug>`, while the template says `wt/<slug>`.

This change removes the guidance from the scaffold. The audit removes it from
repos that already have it, and keeps any rule in it that isn't about worktree
mechanics.

## Outcome

**What you get:**

- **New repos:** a scaffold run asks one git question (where commits land)
  instead of two, and writes no worktree subsection.
- **Repos with a worktree section** (flux, mermaid-office): an audit run lists
  that section as one Outdated finding. Applying it removes the section.
  - It keeps only rules about what reaches `origin`, such as mermaid-office's
    "never push any branch but `main`", which today lives only inside that
    section.
  - Everything else in the section goes, including its safety rules about
    other sessions' servers and uncommitted work. The finding lists each
    dropped sentence.
  - Applying shows the usual diff and asks before writing.
- **Repos without one** (watson, magic-grow): an audit run asks nothing about
  worktrees.

**How to verify:**

- `grep -rnE "Working in a worktree|a2\b|\(b\) Is work|two\*\* questions|wt/<slug>" SKILL.md templates`
  finds nothing. In `references/audit-checklist.md`, "R6b" and "Working in a
  worktree" appear only inside the retired R6b item.
- Run the scaffold's audit in flux. The worktree section appears as one
  Outdated finding. After applying it, `grep -n "Working in a worktree\|worktree-<slug>" CLAUDE.md`
  finds nothing.
- Run the audit in mermaid-office, where "never push any branch but `main`"
  lives only inside the worktree section (`CLAUDE.md:176`). After applying,
  the section is gone, that rule is in the Git workflow section, and the
  base-copy sentence is in *Changelog discipline*.
- Run the audit in watson. No question about worktrees is asked.
- Scaffold a throwaway repo that has `docs/CONCEPT.md` and
  `docs/ARCHITECTURE.md`. Only one git question is asked, and the root
  `CLAUDE.md` has no worktree subsection.

## Key decisions

- **Worktree guidance leaves the template** (diverges). This reverses R6b's
  current stance, which treats a missing subsection as a gap worth asking
  about.
  - The builder skill becomes the one place that defines worktree mechanics.
  - The cost: a worktree started by hand (`claude -w`) loses its written
    guide. The user accepted that, since builders are the only planned users
    of worktrees.
- **The audit removes existing sections** (extends). It reuses the Outdated
  mechanism, the same way the retired per-edit review mandate is handled
  (audit-checklist R5). Leaving the sections in place isn't an option: flux's
  and mermaid-office's landing steps and keep-or-remove question would
  contradict the builder in exactly the sessions it runs.
- **Only rules about what reaches `origin` survive the removal** (diverges).
  This departs, for this one section, from the checklist's "never delete
  custom sections" rule (:198-206).
  - A rule like "never push any branch but `main`" belongs to R6. If it lives
    only inside the worktree section, it moves to the Git workflow section.
  - Everything else in the subsection goes. That includes safety rules about
    other sessions, such as mermaid-office's "that server is not yours to
    kill" and "never run two sessions against the same checkout", and flux's
    "tell the user rather than committing or stashing" someone else's work.
    The builder skill carries those rules itself.
  - The finding lists each dropped sentence, so the diff and confirm show
    them before anything is written.
- **Fresh-worktree setup lists are dropped, not moved** (new).
  - flux lists `dev-data`, `node_modules`, build output and `wwwroot`.
    mermaid-office lists three `npm install`s, `backend/.env` and the SQLite
    database.
  - The builder will discover setup from the main checkout each time. A
    written list goes stale the moment a feature adds a new `.env`.
- **One general fact moves into *Changelog discipline*** (extends). Inside a
  worktree, `docs/changelog/` is the copy from when the worktree was created,
  so a grep that finds nothing there isn't proof. This is true for any
  worktree session, builders included, so it stays as one sentence beside the
  existing "Where `HEAD` is doesn't matter" bullet (template:33).
- **No commit carve-out in `CLAUDE.md`** (reuses). "A skill the user started
  may commit, and its own review answers the gate" goes in the builder skill,
  the same way `/commitandpush` already answers the gate without
  `CLAUDE.md`'s help.
- **Ignore entries for `.claude/worktrees/` stay** (reuses). The builder still
  creates worktrees there, so the audit leaves `.gitignore` and
  `.git/info/exclude` alone.

## Goals

- The scaffold stops writing worktree guidance and stops asking about it.
- An audit cleans existing worktree guidance out without losing other project
  rules.
- No question about worktrees in either the scaffold or the audit flow.

## Non-goals

- **The builder skill:** setup discovery, the commit carve-out, landing and
  removal, and the safety rules about other sessions' servers and
  uncommitted work. That's its own spec.
- **Running the audit on the four repos.** That happens afterwards, per repo,
  with the audit's usual diff and confirm. Run it only once the builder skill
  exists, so flux and mermaid-office aren't left without worktree guidance in
  between.
- **Changing R6's two modes** (Direct to `main`, Branch + PR) or what they say
  about the changelog.
- **The ephemeral worktrees a multi-agent fan-out creates.** The *Multi-agent
  workflows* section (template:163) stays as it is.

## Design

### `templates/CLAUDE-root.md`

- **Delete *Working in a worktree*** (:183-214) whole, including its
  `<Keep this subsection…>` note.
- **Rewrite the Git workflow note** (:169) to cover one choice: keep one mode
  under *Where commits land*. Drop everything about the second choice and how
  the two compose.
- **Add one sentence to *Changelog discipline*,** next to the "Where `HEAD` is
  doesn't matter" bullet (:33): in a worktree you read the base copy, so a grep
  that finds nothing there isn't proof nothing exists. Adapt it from the
  deleted :191.
- **Keep, as they are:**
  - "a commit inside a worktree is still a commit" (:181);
  - the *Owed follow-ups* base-copy note (:106);
  - the fan-out note (:163).

  All three are true without the subsection.

### `SKILL.md`

- **Step 4, Git workflow (:59-67).** Ask one question, not two: (a) where
  commits land.
  - The lead sentence (:59), "ask **two** questions, not one… always ask
    both", becomes a single question.
  - Delete (b) (:63).
  - Change "Neither answer touches…" (:65) to "The answer doesn't touch…".
  - The "where, not when" parenthetical (:67) stays.
- **Step 6** loses **a2** (:112) and the sentence in :127 that makes a2 the
  exception to "don't invent work".
- **The migration-span sentence (:131)** names R6 alone as the section that
  carries the landing sequence.

### `references/audit-checklist.md`

- **R6b becomes a retired item.**

  **R6b — Retired: worktree guidance.**
  - *Outdated looks like:* any *Working in a worktree* subsection, or worktree
    landing rules in any form, including the old three-mode wording ("Direct
    to `main`, parallel work in worktrees").
  - *Fix:*
    - Remove the subsection.
    - Move any rule about what reaches `origin` into R6's section.
    - Move the base-copy sentence into *Changelog discipline*, unless it's
      already there.
    - Drop everything else in the subsection, including setup lists and
      safety rules about other sessions' work. List each dropped sentence in
      the finding.
    - Leave ignore entries alone.
  - *Why:* the builder skill owns worktree mechanics, and a second copy
    contradicts it.
  - *No question:* the file says it all, so this is never asked about.
  - *Absent is OK:* a file with no worktree guidance has no finding here.
- **R6 (:125, :129).** Drop "a separate, additive question assessed under R6b"
  and the "answers no to the worktree question" clause. R6 is now the only
  place a landing sequence gets checked.
- **R3's migration (:75-76, :80, :83, :97).** "R6/R6b" becomes "R6", and the
  `docs/CHANGELOG.md` layout check gets new triggers.
  - The "worktrees not in use" condition (:75) and the "worktrees are in use
    per the a2 answer" trigger (:76) both become a disk check:
    `.claude/worktrees/` exists in the repo.
  - The builder always creates that folder, so its presence shows that
    parallel work lands here. Nothing is asked.
- **R8 (:154).** "unlike R6b" becomes "no user question needed".
- **R3 gains the base-copy sentence** as expected content, matching the
  template. A repo that never had a worktree section, such as watson or
  magic-grow, gets it as a small Missing finding.

## Alternatives considered

- **A "Fresh checkout setup" section in `CLAUDE.md`**, filled per repo. It was
  dropped:
  - It only exists once someone runs the scaffold or the audit.
  - It goes stale when a feature adds a new `.env` or install step.

  The builder discovering setup from the main checkout each time has neither
  problem.
- **Leave existing worktree sections alone and only stop recommending them.**
  Less churn, but flux and mermaid-office would keep landing and keep-or-remove
  rules that contradict the builder in the very sessions it runs.
- **Move the subsection's safety rules into Git workflow** instead of
  dropping them. This came up in the spec review. The user chose to drop them
  with the section, and the builder skill carries them instead.
- **Ask the user whether parallel work happens**, to trigger the changelog
  migration. The audit checks for `.claude/worktrees/` instead, so no
  question is asked.
- **Keep the subsection as an opt-in for hand-made worktrees.** That means two
  definitions of the same mechanics, which is how flux and the template
  drifted apart in the first place.

## Implementation strategy

*Not part of the design — a starting point for whoever builds this.*

- **Single agent, Opus 5.5.** Three files that point at each other (template ↔
  SKILL.md ↔ checklist), with small, interlocking edits. Splitting them would
  only create cross-reference mismatches.
