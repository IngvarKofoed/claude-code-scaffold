---
name: scaffold
description: "Bootstrap a repository's CLAUDE.md scaffolding once `docs/CONCEPT.md` and `docs/ARCHITECTURE.md` already exist. Use this skill when the user says any of: 'scaffold this repo', 'set up CLAUDE.md', 'bootstrap this repo for Claude', 'initialize the agent scaffolding', 'create the CLAUDE.md files', 'wire up the CLAUDE.md', or any phrasing that implies a fresh repo needs its CLAUDE.md files written now that the anchoring docs are done. Also triggers on 'I've written my CONCEPT and ARCHITECTURE, now what?' or 'create root and subtree CLAUDE.md'. Writes root + per-subtree CLAUDE.md from templates in this skill's folder, establishes the per-entry `docs/changelog/` convention, optionally wires git-describe app versioning, prints next steps. When CLAUDE.md files already exist it never overwrites blind: it audits each one against this skill's current recommendations, reports per file what's missing or outdated (e.g. an old review mandate, a section an earlier version of this skill wrote and has since retired or cut down, a required-reading block still carrying the old changelog reading ritual), and asks which files to bring up to date — applying only surgical updates that preserve project-specific content. So also trigger this skill on 'audit my CLAUDE.md', 'are my CLAUDE.md files up to date', 'check my CLAUDE.md against the latest recommendations / scaffold conventions', or 'update CLAUDE.md to the current conventions'."
---

# scaffold

Short bootstrap helper. Run on a repo that already has `docs/CONCEPT.md` and `docs/ARCHITECTURE.md`. Writes the minimal CLAUDE.md scaffolding — root + per-subtree — and nothing else. Stops there. Domain skills, memory, hooks are out of scope.

Works in two modes per target file, decided automatically:
- **Scaffold** — the `CLAUDE.md` doesn't exist yet → write it from the template (steps 4–5).
- **Audit** — the `CLAUDE.md` already exists → never overwrite it blind; check it against this skill's current recommendations and offer surgical updates (step 6).

A single run can do both: e.g. audit an existing root `CLAUDE.md` while scaffolding the subtree files that don't exist yet.

## Writing CLAUDE.md text

Everything this skill writes into a `CLAUDE.md` — a fresh scaffold (steps 4–5) or an audit update (step 6d) — states the current rules as if they were the only rules there have ever been. The session that reads the file has never seen an earlier version of the template or of the file, and current models take a `CLAUDE.md` literally: "there is no summary file to load in its place" or "now that entries no longer share a file" sends it looking for an alternative that doesn't exist in that repo.

Test each sentence, and watch especially for "no longer", "now that", "anymore", "used to", "retired", and "there is no …" naming a mechanism this skill dropped. If the sentence guards against something a fresh session would do on its own — create a `CHANGELOG.md`, start a digest file — keep it, stated as a plain rule. If it only records what an earlier version did, rewrite it as the current rule. `references/audit-checklist.md` names retired schemes because detecting them is its job; that history goes in the finding shown to the user, never in the text written to the file.

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

Look in `docs/ARCHITECTURE.md` first — most architecture docs name the major subtrees in a "Project Structure" section (or equivalent: "Layout", "Repository structure", a folder tree, etc.). If you can extract subtree paths from there, present the list back to the user for confirmation: *"I see these subtrees in ARCHITECTURE.md: `src/backend`, `src/frontend`, `src/e2e`. Each gets a `CLAUDE.md` if it has rules of its own. Anything to add or remove?"*

If ARCHITECTURE doesn't name subtrees, fall back to asking the user directly in one question. Typical examples: `src/backend`, `src/frontend`, `src/e2e`, `services/api`, `packages/cli`.

A subtree gets its own CLAUDE.md only when it has rules of its own: what its tests must never touch, a check that must run from elsewhere, a verification step, a rule about its code. Step 5 decides that per subtree, after testing each line. A small single-language project may have none; that's fine, skip step 5 in that case.

### 4. Partition target files, then write the ones that don't exist

The target `CLAUDE.md` paths are the repo root plus each subtree from step 3. Split them in two:

- **Doesn't exist yet** → scaffold from the template (this step for root, step 5 for subtrees).
- **Already exists** → do **not** write here. Set it aside for the audit pass (step 6), which runs after the scaffold writes. Collecting all existing files first means the audit can present one consolidated "which to update" prompt instead of interrupting per file.

For the root `CLAUDE.md`, if it doesn't exist:

Read the template at `templates/CLAUDE-root.md` in this skill's folder.

Fill placeholders:
- **Project name** — read it from `package.json`, `pyproject.toml`, `*.csproj`, `Cargo.toml`, or ask the user.
- **Git workflow** — ask one question. It can't be inferred, so always ask it.

  **Where do commits land?** Two options: directly to `main`, or branch and open a PR per change. Keep the matching bullet under *Commits* and delete the other. Direct-to-`main` asks whether to push after each commit (leave the push line out unless the user wants it); Branch+PR needs the branch-naming convention and whether `main` is protected.

  **The answer doesn't touch the *Changelog* section.** The entry file is written the same way on `main`, on a branch and in a worktree, so nothing there is mode-dependent. Don't add a conditional to it.

  (This overrides only the *where* half of the agent's generic default, "branch first". The *when* half, "commit only when asked", is Claude Code's own default, so the template doesn't restate it — don't add a line telling the agent to commit when a change is done.)

Write to `<repo-root>/CLAUDE.md`.

### 5. Write subtree `CLAUDE.md` files

For each subtree path from step 3 **whose `CLAUDE.md` doesn't exist yet** (existing ones were set aside in step 4 for the audit in step 6):

1. **Identify the stack** — language and major frameworks for this subtree. Look in `docs/ARCHITECTURE.md` for descriptions, and at the subtree's package manifest if there is one (`package.json`, `*.csproj`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`).

2. **Read `templates/CLAUDE-subtree.md`** in this skill's folder.

3. **Fill the template from the project:**
   - **Testing** — read the test helpers (fakes, mocks, temp repos, a test database) and the manifest's scripts, and write only the four kinds of line the template names: one never-touch line for each real resource the code uses (an external API, a repo, a database, the user's data folder), naming the fake or temp copy tests use instead; a check the test script doesn't run, or one that must run from elsewhere (a type check, from the repo root for a package others import); a script to use where the bare tool skips a pre-step (`npm run typecheck`, never `tsc --noEmit`); a coverage rule the tests already follow.
   - **Verification workflow** — UI subtrees only. Fill step 1 with the exact dev command, the setting that points it at scratch data (read the server's config for a database path or data directory), and what it opens without that setting. Before writing it, confirm Playwright MCP is actually installed and connecting; if it's missing, or only available headed (e.g. bundled by a plugin), offer to install it headless per `references/playwright-mcp.md`. Run that check once per run, not per subtree.
   - **Rules** — rules about this subtree's code that `docs/ARCHITECTURE.md` doesn't state: an action or a location the doc doesn't give. Every session loads that doc, so a rule copied from it is a second copy that drifts.
   - **No list of tools or skills.** A session already has every tool installed on the machine, and a skill runs when its own description matches the work. A mandate repeats that, makes a skill run where it doesn't fit, and fails for anyone who hasn't installed it. A tool appears only inside a step that uses it, as Playwright MCP does in the verification workflow.

4. **Test every line, then decide whether the file exists.** Keep a line only if a session that has read `docs/ARCHITECTURE.md`, the manifest and the folder would act differently without it; the template's closing instructions list what fails. Where a rule is a mechanical check no test enforces — search the diff for a forbidden pattern — leave it out and name it in step 10's summary as a test worth adding; this skill doesn't write code. Leave out a section with no surviving line. A subtree with no surviving line gets no file: name it in step 10's summary, so the user knows it was considered.

### 6. Audit existing `CLAUDE.md` files (if any)

For every `CLAUDE.md` set aside in step 4 because it already existed, run an audit instead of overwriting. The point is to bring an older or hand-written file up to this skill's *current* recommendations without clobbering the project-specific content someone added.

**a. Audit each file against the checklist.** Read `references/audit-checklist.md` in this skill's folder — it's the rubric (what every root and subtree `CLAUDE.md` should contain, the *why* behind each item, and what an outdated version looks like). For each existing file, classify every rubric item as **OK / Missing / Outdated / Stale**. Satisfied items aren't findings — only surface gaps. For existing subtree files you'll need the same stack/test-framework facts step 5 gathers (read the manifest + architecture doc) so you can judge S3 and S6 fairly.

One rubric item can't be judged from the files at all: R3's **old-layout check**, `test -f` on `docs/CHANGELOG.md`, `docs/CHANGELOG-archive.md` and `docs/CHANGELOG-DIGEST.md`. Run it once, here. The skill no longer converts an old layout: a hit is reported as one finding and the files are left exactly as they are.

**b. Present findings, grouped by file.** One scannable line per finding: the item, its classification, and a few words of why it matters — e.g.:

> **`CLAUDE.md` (root)** — 3 findings
> - *Outdated:* reviews after every non-trivial edit — current convention is one review point, at the commit, so the accumulated diff is read as one change instead of a pass per prompt.
> - *Outdated:* the read-first block still carries the changelog reading ritual (newest 10, a delegated digest) — cut to `ls` plus a grep before touching an area.
> - *Outdated:* still has the retired *Owed follow-ups* section — removed; `docs/CLAUDE_TODO.md` itself is left alone.
>
> **`src/web/CLAUDE.md`** — 1 finding
> - *Missing:* no browser-driven verification workflow for a UI subtree.
>
> **`src/api/CLAUDE.md`** — OK, nothing to update.

If nothing is outdated anywhere, say so plainly and move on — don't invent work.

**c. Ask which files to update.** One consolidated prompt offering the files that have findings — the user replies `all`, `none`, or a subset. Let them narrow within a file too ("just the changelog fix in root"). A file with no findings isn't offered.

**d. Apply surgically to the chosen files.** Follow the "Applying updates" rules in `references/audit-checklist.md`: add Missing sections (filled from the project, not raw placeholders), rewrite Outdated sections to the current intent and length while keeping surrounding project-specific wording — stating the current rule only, per *Writing CLAUDE.md text* above — remove the sections the template has retired, fix Stale items, and **never delete custom sections the template never had**. Show the diff and confirm before writing each file. Files the user didn't pick are left exactly as-is.

### 7. Create nothing for the changelog

Counter-intuitive enough to be a step rather than an omission: a fresh scaffold creates **no** changelog file of any kind.

- **Don't create `docs/changelog/`.** The first entry creates it. Git can't track an empty directory, and a `.gitkeep` in a folder that will be permanently populated is dead weight nobody ever deletes.
- **Don't create a digest, decision log, or any other curated summary** — not `docs/CHANGELOG-DIGEST.md`, not under some other name, not even empty. History is reached by `ls docs/changelog` plus a grep over the folder, which the root CLAUDE.md tells every session to run before touching an area (step 4 wrote it). A curated file is one that every change appends to and every compaction rewrites, which is exactly the collision the per-entry folder exists to remove; the append conflicts trivially, but the rewrite is a real semantic conflict and it falls due precisely when there is enough parallel work to make it expensive.
- **Don't create `docs/CHANGELOG.md`.** The convention is the folder.
There is no dangling-reference problem to design around, because none of this is `@`-referenced: the changelog read is an *action*, and `ls` against a missing folder just means no entries yet.

**If the repo already carries a `docs/CHANGELOG.md`, `docs/CHANGELOG-archive.md` or `docs/CHANGELOG-DIGEST.md`, name it in step 10's summary — even on a pure scaffold run**, where step 6's old-layout check never runs. The `CLAUDE.md` this run wrote doesn't read any of them, so the history in them is now reached by nothing. Converting it is the user's call: never edit, move or delete those files.

### 8. Offer git-describe app versioning (optional)

Ask first; skip entirely if the user declines. This only pays off for a **deployed application** with a build/CI pipeline — a library, a one-off script, or a repo with no build step doesn't need it.

**The scheme.** Version the app from git, not a hand-bumped number. One annotated tag (`v0.1.0`) plus `git describe --tags --long --always --dirty` (e.g. `v0.1.0-32-g98af7ae`) is the version. New releases are just new annotated tags — no code change. (`--long` keeps the `-g<sha>` suffix even when HEAD is exactly on a tag — without it an exact tag prints just `v0.1.0` and the build loses its commit SHA; `--always` degrades to a bare short hash when there's no tag at all; `--dirty` flags uncommitted local builds.)

**The mental model that makes it work.** A deployed artifact — a container, a published binary, a built frontend bundle — has no `.git` and no git binary. So the version is **resolved once at build time, inside the checkout, and baked into the artifact**. The running app never shells out to git; it only reads the baked-in value. This is exactly how the git-versioning build tools work (MinVer, setuptools_scm, Nerdbank.GitVersioning), and every recipe in the reference follows that same shape — always baked in, never resolved at runtime.

**The universal runtime contract.** The app reads its version from whatever the build baked in for its stack — a stamped assembly/manifest, a generated version module, or an env var set at build — and falls back to `"unknown"` only when the bake step never ran. `git describe` runs in exactly one place: the build step (always inside the checkout, in CI and locally alike). Never make the running app resolve git itself.

If the user opts in:

1. **First tag** — if the repo has no tags yet, create the initial annotated one: `git tag -a v0.1.0 -m "Initial release marker"`. `git describe` needs at least one tag (or `--always`) to produce anything.

2. **Wire the ecosystem idiom.** Detect the stack — per subtree if they differ — and read the matching section of `references/app-versioning.md` in this skill's folder. It has copy-pasteable build-time-inject + runtime-read recipes for Node.js (backend & frontend), .NET, Python, Go, Rust, JVM, and containers. Apply the idiomatic mechanism (.NET MinVer, Python setuptools_scm, Go `-ldflags`, frontend bundler `define`, …) rather than bolting a generic runtime `git describe` onto a stack that already has a first-class one.

3. **The CI gotcha that bites every stack** — CI checkouts default to a shallow clone with no tags, so `git describe --tags` silently degrades to a bare hash and builds lose their version. Whatever the pipeline, make the checkout fetch full history and tags (GitHub Actions: `fetch-depth: 0`; otherwise `git fetch --tags --unshallow`). Flag this in any build config you touch.

### 9. Offer to install the companion skill (optional)

The root CLAUDE.md this skill writes never depends on a companion skill, but its commit flow pairs well with one. Same reasoning as Playwright MCP (`references/playwright-mcp.md`): a mandate that names a tool is worth more when the tool is actually there, so check, and offer — don't assume, and don't install unasked.

**The repo name and the skill name differ** — the repo is prefixed `claude-code-`, the skill is not. Install directory and skill name are always the short form:

| Skill | Install as | What the scaffolding uses it for | Repo |
| --- | --- | --- | --- |
| `git` | `~/.claude/skills/git/` | Commit/push delegated to a cheap Haiku subagent, keeping the main session's context and cost down. | `https://github.com/IngvarKofoed/claude-code-git` |

**a. Detect whether it's already there.** The skill counts as installed if it appears in your own available-skills listing, or if `~/.claude/skills/git/` exists — the **short** name, not the repo name. **If it's installed, skip this step in silence** — no question, no mention in the summary.

**b. Ask only if it's missing.** One `AskUserQuestion` question. Options: **install** (user-level, `~/.claude/skills/`, available in every repo) or **skip** (the agent commits directly without it). Never install without an explicit pick.

**c. Install user-level, under the short name.** Always name the target directory explicitly, or git derives it from the URL and you get `~/.claude/skills/claude-code-git/` — a folder whose name doesn't match the skill:

```bash
git clone https://github.com/IngvarKofoed/claude-code-git ~/.claude/skills/git
```

`skill-manager:add` is the nicer route when that plugin is present, since it registers the skill for auto-update — but confirm afterwards that the installed folder is the short name, and rename it if not.

**d. Note the reload.** A freshly installed skill isn't callable in the current session until skills reload — tell the user to run `/reload-skills` or restart Claude Code. Until then the agent commits directly, which is fine.

Install user-level only. Don't offer to clone the skill into the repo — a nested clone can't be committed as-is, and a personal skill inside a shared repo decides the whole team's toolchain.

If the clone fails (no network, no access to a private repo), say so in one line and move on. The scaffolding works without the skill.

### 10. Print next steps

Print a short summary listing:

- Files created (scaffolded fresh), files updated (audited in step 6 and brought current), and files left alone (existing, no findings or not picked) — with paths.
- Reminders:
  - **Subtree CLAUDE.md files** hold only lines that passed step 5's test, filled in from the test helpers, the manifest and the architecture doc. List the subtrees that got no file because no line passed. Skim and adjust.
  - **Mechanical checks worth turning into tests** — any rule step 5 or the audit found that asks a session to search for a pattern by hand, named with the subtree. A test enforces it for every session; a sentence only asks.
  - **If you installed or reconfigured Playwright MCP during this run** (`references/playwright-mcp.md`), restart Claude Code or reconnect via `/mcp` before relying on the UI verification workflow — the new `mcp__playwright__*` tools aren't loaded in the current session yet.
  - **Review happens once, at the commit — not after every edit.** Deliberately: a pass per prompt only ever sees its own turn's edits and costs more than it catches. When you ask the agent to commit and the working tree holds non-trivial changes, it asks first, with two options: **`/code-review high --fix`** (broad coverage, repairs applied to the working tree, recommended for the accumulated diff this gate usually sees) or **commit now**. The agent runs your choice itself. `/code-review` ships with the harness, so nothing has to be installed for the gate to work.
  - **What the gate does *not* defer** — tests, build, and each subtree's own verification workflow still run per change. Only the review waits for the commit, and the question is skipped for a trivial diff.
  - **The changelog read is two actions and no `@`-reference** — nothing can `@`-reference "the files of a folder". Before touching an area, the session lists the folder and greps it. Either can be skipped, and it fails quietly when it is — the grep especially, since nothing signals a decision you never thought to look for. That is the accepted cost of having no curated digest: such a file is one every change appends to and every compaction rewrites, which is the collision the per-entry folder exists to remove. If you want the read deterministic, a session-start hook is the way (`update-config`) — out of this skill's scope, so yours to add.
  - For project-specific domain skills (a design system, security rules, naming conventions), use `/skill-creator`, and give the skill a description that names the work it's for. The description, not a line in `CLAUDE.md`, is what makes a session invoke it.

Stop. Do not start building domain skills or expanding the docs — those are separate decisions for the user.

## Safety

- Never overwrite an existing `CLAUDE.md` blind. An existing file goes through the audit pass (step 6): updates are surgical, shown as a diff, and applied only to files the user picks. Custom sections the template never had are never deleted. A subtree file is deleted only when the audit's updates leave no line in it (rubric S7), shown and confirmed like any other change.
- Never modify `CONCEPT.md` or `ARCHITECTURE.md`.
- Never modify, move or delete `docs/CHANGELOG.md`, `docs/CHANGELOG-archive.md` or `docs/CHANGELOG-DIGEST.md`. The skill no longer converts an old changelog layout; it reports one (step 6a, step 7).
- Never create, edit, move or delete `docs/CLAUDE_TODO.md` or `TODO.md`. The skill has no convention for either any more, and removing a retired section about them from a `CLAUDE.md` (R8, R9) leaves the files themselves exactly as they are.
- Never modify a landed file in `docs/changelog/`. Its path is its address; renaming or deleting one breaks every citation to it and produces no conflict to catch.

## Scope

**In scope:** writing CLAUDE.md scaffolding, auditing and surgically updating existing CLAUDE.md files against the current recommendations — including cutting the sections earlier versions of this skill wrote and have since retired or shortened (step 6, rubric in `references/audit-checklist.md`), reporting an old changelog layout without converting it (step 6a, step 7), optionally wiring git-describe app versioning (step 8, only if the user opts in — per-ecosystem recipes live in `references/app-versioning.md`), checking for the `git` companion skill and offering to install it user-level if it's missing (step 9, opt-in), and — for UI subtrees only — verifying Playwright MCP is installed and offering to install it headless if it isn't (`references/playwright-mcp.md`).

**Out of scope:** creating CONCEPT or ARCHITECTURE (the user does this), building domain skills (use `/skill-creator`), configuring hooks and general MCP-server or plugin management beyond the narrow Playwright-for-UI convenience above (use `update-config`), writing to user-side memory (per-user, not repo-side).
