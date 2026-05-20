---
name: scaffold
description: "Bootstrap a repository's CLAUDE.md scaffolding once `docs/CONCEPT.md` and `docs/ARCHITECTURE.md` already exist. Use this skill when the user says any of: 'scaffold this repo', 'set up CLAUDE.md', 'bootstrap this repo for Claude', 'initialize the agent scaffolding', 'create the CLAUDE.md files', 'wire up the CLAUDE.md', or any phrasing that implies a fresh repo needs its CLAUDE.md files written now that the anchoring docs are done. Also triggers on 'I've written my CONCEPT and ARCHITECTURE, now what?' or 'create root and subtree CLAUDE.md'. Writes root + per-subtree CLAUDE.md from templates in this skill's folder, seeds an empty `docs/CHANGELOG.md`, prints next steps. Refuses to overwrite existing CLAUDE.md files — shows a diff and asks instead."
---

# scaffold

Short bootstrap helper. Run once on a repo that already has `docs/CONCEPT.md` and `docs/ARCHITECTURE.md`. Writes the minimal CLAUDE.md scaffolding — root + per-subtree — and seeds the changelog. Stops there. Domain skills, memory, hooks are out of scope.

## Workflow

Each step is a hard gate. If a step's check fails, stop and tell the user — do not silently skip.

### 1. Verify anchoring docs exist

Check for `docs/CONCEPT.md` and `docs/ARCHITECTURE.md` in the target repo.

If either is missing, refuse to proceed. The user needs to write those first — they're the contract the CLAUDE.md scaffolding references, and without them the scaffolding has nothing to anchor on. Brief guidance on what each should cover:

- `CONCEPT.md` — what the project does at the *domain* level: entities, lifecycles, vocabulary, user-facing behavior. No class names, table names, or stack details — those belong in ARCHITECTURE.
- `ARCHITECTURE.md` — *how* it's built: stack, system topology, internal layers/components, data-model topology (not column-level), inter-process protocols, project structure, testing, deployment.

Neither is templatable — let the structure emerge from the domain.

### 2. Check doc sizes

Run `wc -l docs/CONCEPT.md docs/ARCHITECTURE.md`.

If either is over ~600 lines, surface this and suggest splitting before continuing. The agent reads these docs at session start; long docs get skimmed instead of read. A common split: pull the column-level data model out of ARCHITECTURE.md into `docs/DATABASE.md`. Let the user decide whether to continue as-is or split first.

### 3. Identify subtrees

Look in `docs/ARCHITECTURE.md` first — most architecture docs name the major subtrees in a "Project Structure" section (or equivalent: "Layout", "Repository structure", a folder tree, etc.). If you can extract subtree paths from there, present the list back to the user for confirmation: *"I see these subtrees in ARCHITECTURE.md: `src/backend`, `src/frontend`, `src/e2e`. Each gets a scoped CLAUDE.md. Anything to add or remove?"*

If ARCHITECTURE doesn't name subtrees, fall back to asking the user directly in one question. Typical examples: `src/backend`, `src/frontend`, `src/e2e`, `services/api`, `packages/cli`.

A subtree warrants its own CLAUDE.md when the rules genuinely differ — different language, different tools, different test framework. A small single-language project may have none; that's fine, skip step 5 in that case.

### 4. Write root `CLAUDE.md`

Read the template at `templates/CLAUDE-root.md` in this skill's folder.

Fill placeholders:
- **Project name** — read it from `package.json`, `pyproject.toml`, `*.csproj`, `Cargo.toml`, or ask the user.
- **Nested guidance pointers** — replace the example lines with the actual subtree paths from step 3, each with a one-line scope description (ask the user for the scope line if you can't infer it).

Write to `<repo-root>/CLAUDE.md`. **Refuse to overwrite** if the file already exists. Instead, show the user a diff against the template and ask how to proceed.

### 5. Write subtree `CLAUDE.md` files

For each subtree path from step 3, read `templates/CLAUDE-subtree.md` in this skill's folder and write `<subtree>/CLAUDE.md`.

If the subtree directory doesn't exist yet, create it.

The subtree template leaves placeholders for required tools, required skills, test framework, and subtree-scoped rules. **Do not try to fill those in** — the user knows their stack better than you do. Leave the placeholders visible so the user knows what to fill in.

Same overwrite rule: if `<subtree>/CLAUDE.md` exists, do not overwrite; show a diff and ask.

### 6. Seed `docs/CHANGELOG.md`

If `docs/CHANGELOG.md` doesn't exist, create it with this five-line header so the agent has somewhere to write entries:

```markdown
# Changelog

Each entry is numbered with a monotonically increasing integer. Append new entries to the end. Never reuse or reorder numbers. Numbers are globally unique across this file and any future `CHANGELOG-archive.md` — never reused.

```

The full discipline rule lives in root CLAUDE.md (step 4 wrote it). The file itself just needs to exist.

If `docs/CHANGELOG.md` already exists, leave it alone.

### 7. Print next steps

Print a short summary listing:

- Files created / files left alone (with paths).
- Two reminders:
  - The `simplify` skill is already globally available — root CLAUDE.md already mandates it after non-trivial edits, so nothing to install.
  - For project-specific domain skills (a design system, security rules, naming conventions), use `/skill-creator`. Add the required-skill mandate to the relevant subtree CLAUDE.md once the skill exists.

Stop. Do not start filling subtree placeholders, building domain skills, or expanding the docs — those are separate decisions for the user.

## Safety

- Never overwrite an existing `CLAUDE.md` without explicit user confirmation.
- Never modify `CONCEPT.md` or `ARCHITECTURE.md`.
- Never modify `CHANGELOG.md` if it already exists.

## Scope

**In scope:** writing CLAUDE.md scaffolding, seeding `docs/CHANGELOG.md`.

**Out of scope:** creating CONCEPT or ARCHITECTURE (the user does this), building domain skills (use `/skill-creator`), configuring hooks (use `update-config`), writing to user-side memory (per-user, not repo-side).
