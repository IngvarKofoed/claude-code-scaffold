# Write only current rules into CLAUDE.md

A prompt audit of claude-hive, a repo this skill scaffolded, found three kinds
of text in the generated `CLAUDE.md` files that mislead the models reading
them. Claude Opus 5.5 and later take a `CLAUDE.md` literally, so every
sentence in a generated file has to state a current rule or the reason for
one.

- **History-only wording.** The root template described some rules by
  contrast with earlier versions of itself ("There is no summary file to load
  in its place", "now that entries no longer share a file", "there is no
  end-of-turn nudge…"). A session in a scaffolded repo has never seen those
  versions, so it reads each one as a pointer to an alternative and goes
  looking for it.
- **`claude-api` mandated for Agent SDK code.** The catalog said to require
  `claude-api` in subtrees that import the Anthropic SDK. claude-hive's
  backend is built on the Claude Agent SDK, and its `CLAUDE.md` now sends
  every SDK change through a skill that says it doesn't cover that SDK.
- **Loading recipes for deferred tools.** Subtree files repeated "load it
  with `ToolSearch select:<name>`". The harness already tells every session
  how to load deferred tools.

## Outcome

**What you get:**

- **New scaffolds.** The root `CLAUDE.md` states each rule on its own terms.
  Subtree files name each required tool and why work there needs it, with no
  loading recipe. A subtree built on the Agent SDK is offered a rule to check
  the SDK's type definitions or ask `claude-code-guide`, not `claude-api`.
- **Audits of existing repos.** The three phrases come up as Outdated
  findings under R1, R3 and R5, each with the rewrite offered. A loading
  recipe is a minor S2 finding. `claude-api` mandated for an Agent SDK
  subtree is an S5 finding.

**How to verify:**

- `grep -nEi "no longer|now that|anymore|used to|retired|there is no" templates/`
  finds only `CLAUDE-root.md:27` and `:36` (kept guards), `:51` (the sample
  changelog entry's text), `:99` (a current reason), and "there is no"
  matching "there is nothing" (`:16`, `:178`) or "there is noise" (`:142`).
  `grep -n ToolSearch templates/` finds nothing.
- A report-only audit of claude-hive flags `CLAUDE.md`'s three phrases and
  the `claude-api` line in `packages/backend/CLAUDE.md`.

## Key decisions

- **One test for every sentence** (new). If a sentence guards against
  something a fresh session would do on its own, keep it with its reason. If
  it only records what an earlier template version did, rewrite it as the
  current rule. Under that test these stay in the root template:
  - `:27` "There is no single changelog file and there are no entry
    numbers" stops a session from creating a `CHANGELOG.md`.
  - `:36` "There is no digest and no curated decision list — deliberately…"
    stops it from starting a digest, and gives the reason.
  - `:31` "with no digest carrying it" is the reason for the back-citation
    rule. It rests on `:36`'s current fact, not on the old digest.
  - `:33` "with nothing staged or moved at any point" and `:178` "nothing to
    stage and nothing to remember at hand-off" stop a session from staging
    entries for a later compile, the way changesets or towncrier work.
  - `:126` "Do not review after every edit" and `:164` "This doesn't earn an
    extra pass on return" stop a session's own habit of reviewing as it goes.
  - `:51` "can no longer silently renew" is the sample changelog entry. An
    entry states what changed, so it is supposed to describe a delta.
- **The rule lives in `SKILL.md`**, as a short section *Writing CLAUDE.md
  text* that covers both scaffold writes (steps 4–5) and audit updates (step
  6d). The checklist keeps naming retired schemes, because detecting them is
  its job. Its *Applying updates* list gains a bullet saying that history
  goes in the finding, never in the file.
- **History-only wording is Outdated even where the intent is met**
  (extends). This sits beside, not against, the checklist's rule that a
  recommendation is judged by intent, not wording. The sentence is a finding
  because of what a literal reader does with it, not because it differs from
  the template. R3 names the two look-alike sentences (`:27`, `:36`) as not
  this finding, so the audit doesn't flag the guards.
- **R3 migration step 5's pointer is keyed on the marker** (changes). It
  used to say "entries predating the migration carry synthetic dates". A
  session never saw the migration, but it can see `<!-- entry N -->`, so the
  sentence it writes now says an entry carrying that marker has a synthetic
  date.
- **`claude-api` is for the Claude API and Managed Agents only** (changes).
  Checked on 2026-11-26 against the `claude-api` skill bundled with Claude
  Code 2.1.332: "This skill covers the Claude API and Managed Agents
  (options 1-3); it does not generate Claude Agent SDK code." The built-in
  `claude-code-guide` agent's description lists "Claude Agent SDK - building
  custom agents". The catalog's `claude-api` entry now says what it doesn't
  cover. A new entry, *Claude Agent SDK subtrees (no skill)*, requires
  checking the SDK's own type definitions in the installed package, or
  asking `claude-code-guide`, before writing SDK calls. `SKILL.md` and the
  checklist had no copy of the mapping. The checklist gains an S5 note, so
  an audit finds the wrong mandate in repos already scaffolded.
- **Deferred tools: name and reason, no loading recipe** (changes; the
  audit finding was low-confidence).
  - The harness tells every session which tools are deferred and how to
    load them, so the recipe adds nothing a session doesn't already have.
  - Whether a given tool is deferred, and the loader's query syntax, change
    between Claude Code versions. A literal reader follows a stale recipe.
  - What the line is for is naming the tool and saying why work here needs
    it (`LSP` instead of grepping for symbols). Both stay.
  - Changed: the subtree template's tool line, `SKILL.md` step 5's "Quote
    the load instructions" (`:78` before this change, the line that put the
    recipes in generated files), and S2. S2 calls an existing recipe a
    *minor* Outdated finding, matching the low confidence.
  - `SKILL.md`'s "Deferred tools are discoverable via `ToolSearch` queries"
    stays. It tells the scaffolding session how to discover what's
    installed, and nothing copies it into a file.
- **`/reload-skills` stays** (reuses). It is a registered command in Claude
  Code 2.1.332 (`name:"reload-skills"`, "Pick up skills added or changed on
  disk during this session", not hidden). This was checked in the installed
  binary, not in the docs or with `claude-code-guide`.

## Goals

- Nothing the skill writes into a `CLAUDE.md` carries the history of the
  skill's own templates.
- Audits find that wording, the wrong `claude-api` mandate and loading
  recipes in repos scaffolded before this change.

## Non-goals

- Editing any project's `CLAUDE.md`. claude-hive's are being fixed
  separately.
- Editing any other skill.
- Restructuring or shortening the templates. Their reasons and rejected
  alternatives are context, and they stay.

## Design

- **`templates/CLAUDE-root.md`.** `:12` drops "There is no summary file to
  load in its place:". `:29` ends "because nothing else bounds it: each
  entry has a file to itself." `:139` ends at "stops to ask."
- **`templates/CLAUDE-subtree.md`.** The tool line loses its
  `<Load instructions if deferred…>` placeholder.
- **`SKILL.md`.** Adds the *Writing CLAUDE.md text* section before
  *Workflow*. Step 5 says to name each tool and its reason and to leave out
  how to load it. Step 6d points at the new section.
- **`references/audit-checklist.md`.**
  - Adds a paragraph under *How to classify*: history-only wording is
    Outdated, and why that doesn't conflict with judging by intent.
  - Adds the three phrases to the *Outdated looks like* notes of R1, R3 and
    R5. R3's note also names the two sentences that are not this finding.
  - R5's description gains a note that its "no per-edit review pass, and no
    end-of-turn nudge" is for the auditor and isn't copied into a file.
  - R3 migration step 5 keys the pointer on the marker.
  - S2 drops "load instructions quoted" and gains the minor recipe finding.
  - S5 gains the wrong-mandate finding.
  - *Applying updates* gains *Write the current rule, not its history*.
- **`references/subtree-skills.md`.** Narrows the `claude-api` entry and adds
  the Agent SDK entry.

## Alternatives considered

- **Delete every "there is no…" sentence.** Rejected: `:27` and `:36` stop a
  fresh session from creating a `CHANGELOG.md` or a digest, defaults it
  would otherwise pick.
- **Keep the loading recipes for sessions that don't know about deferred
  tools.** Rejected: every session already gets the harness's own loading
  instructions, and a recipe for a tool that isn't deferred in some version
  is an instruction that misleads.
- **Keep `claude-api` for Agent SDK subtrees, since its text compares the
  Agent SDK with the API.** Rejected: the comparison exists to send the
  reader elsewhere, and a mandate would route every SDK change through a
  skill that won't write SDK code.
- **Point Agent SDK subtrees at the docs page** (`code.claude.com/docs/en/agent-sdk`,
  which the `claude-api` skill names). Not chosen: the installed package's
  type definitions match the version the subtree actually runs, and
  `claude-code-guide` reads the docs when that is needed.
- **Replace `/reload-skills` with "restart Claude Code".** Not needed: the
  command exists.

## Implementation strategy

*Not part of the design — a record of how it was built.*

- **Single agent, Opus 5.5**, for the edits. The template, `SKILL.md` and
  the checklist point at each other, so one author keeps them consistent.
- **One fresh subagent** ran the report-only audit of claude-hive, so the
  check reads the edited checklist without the editor's reading of it.
