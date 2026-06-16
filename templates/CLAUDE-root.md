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
- Keep entries short and to the point — one line where possible, a few lines at most. Focus on *what* changed, not how.
- Write the entry as part of the same change. Do not batch multiple changes into one entry, and do not skip entries.
- When a phase/increment completes, its per-task entries move to `docs/CHANGELOG-archive.md`, leaving only the milestone summary in `docs/CHANGELOG.md`. Numbers are globally unique across both files — never reuse one that already appears in either.

## Nested guidance

Each `src/` subtree (or service / package / area) has its own `CLAUDE.md` with scoped tool/skill rules:

- `src/<subtree-a>/CLAUDE.md` — <one-line scope>
- `src/<subtree-b>/CLAUDE.md` — <one-line scope>

## After making changes

After a non-trivial edit, invoke the **`code-review`** skill with the `high --fix` arguments to review the touched code for correctness, reuse, clarity, and efficiency, then apply every finding to the working tree automatically. It does not auto-trigger — you must invoke it explicitly.

Once the fixes are applied, report what changed:

1. **Group the applied fixes by severity** — blockers (correctness bugs, data loss, security), should-fix (clear improvements, missed reuse), nits (style, naming, minor clarity).
2. **Summarize each bucket in one line** so the user can see what was fixed without expanding every finding.
3. Do not stop to ask which to fix — all findings are fixed by default. The user can review the diff and revert anything they disagree with.

<!-- Add additional sections below as the project develops:
  - Project-specific forcing rules (e.g., "Check in with the user before making CSS / layout / UX changes")
  - Destructive-operation guidance if the agent's defaults aren't enough
  - Naming conventions, code-organization rules
-->
