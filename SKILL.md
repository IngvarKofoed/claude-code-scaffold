---
name: scaffold
description: "Bootstrap a repository's CLAUDE.md scaffolding once `docs/CONCEPT.md` and `docs/ARCHITECTURE.md` already exist. Use this skill when the user says any of: 'scaffold this repo', 'set up CLAUDE.md', 'bootstrap this repo for Claude', 'initialize the agent scaffolding', 'create the CLAUDE.md files', 'wire up the CLAUDE.md', or any phrasing that implies a fresh repo needs its CLAUDE.md files written now that the anchoring docs are done. Also triggers on 'I've written my CONCEPT and ARCHITECTURE, now what?' or 'create root and subtree CLAUDE.md'. Writes root + per-subtree CLAUDE.md from templates in this skill's folder, seeds `docs/CHANGELOG-DIGEST.md` for the per-entry `docs/changelog/` convention, optionally wires git-describe app versioning, prints next steps. When CLAUDE.md files already exist it never overwrites blind: it audits each one against this skill's current recommendations, reports per file what's missing or outdated (e.g. an old review mandate, a changelog section still on the retired single-file numbered scheme), and asks which files to bring up to date — applying only surgical, additive updates that preserve project-specific content. So also trigger this skill on 'audit my CLAUDE.md', 'are my CLAUDE.md files up to date', 'check my CLAUDE.md against the latest recommendations / scaffold conventions', or 'update CLAUDE.md to the current conventions'."
---

# scaffold

Short bootstrap helper. Run on a repo that already has `docs/CONCEPT.md` and `docs/ARCHITECTURE.md`. Writes the minimal CLAUDE.md scaffolding — root + per-subtree — and seeds the changelog digest. Stops there. Domain skills, memory, hooks are out of scope.

Works in two modes per target file, decided automatically:
- **Scaffold** — the `CLAUDE.md` doesn't exist yet → write it from the template (steps 4–5).
- **Audit** — the `CLAUDE.md` already exists → never overwrite it blind; check it against this skill's current recommendations and offer surgical updates (step 6).

A single run can do both: e.g. audit an existing root `CLAUDE.md` while scaffolding the subtree files that don't exist yet.

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

### 4. Partition target files, then write the ones that don't exist

The target `CLAUDE.md` paths are the repo root plus each subtree from step 3. Split them in two:

- **Doesn't exist yet** → scaffold from the template (this step for root, step 5 for subtrees).
- **Already exists** → do **not** write here. Set it aside for the audit pass (step 6), which runs after the scaffold writes. Collecting all existing files first means the audit can present one consolidated "which to update" prompt instead of interrupting per file.

For the root `CLAUDE.md`, if it doesn't exist:

Read the template at `templates/CLAUDE-root.md` in this skill's folder.

Fill placeholders:
- **Project name** — read it from `package.json`, `pyproject.toml`, `*.csproj`, `Cargo.toml`, or ask the user.
- **Nested guidance pointers** — replace the example lines with the actual subtree paths from step 3, each with a one-line scope description (ask the user for the scope line if you can't infer it).
- **Git workflow** — ask **two** questions, not one, because these are independent axes that compose rather than a single either/or. Neither can be inferred, so always ask both.

  **(a) Where do commits land?** Two options: directly to `main`, or branch and open a PR per change. Keep the matching mode under *Where commits land* and delete the other. Direct-to-`main` needs the push policy; Branch+PR needs the branch-naming convention and whether `main` is protected.

  **(b) Is work here ever isolated in a git worktree?** Yes/no — and *yes* is compatible with **either** answer to (a). This is the question that used to be folded into (a) as a third mode, which was wrong: a repo that normally commits straight to `main` and *sometimes* moves a piece of work into a worktree wants both behaviours, selected per session by where the session actually is, not one convention chosen once at scaffold time. On *yes*, keep the *Working in a worktree* subsection — and trim it to the mode chosen in (a), because it is written to serve both: drop the *Landing under `Direct to main`* steps and their keep/remove references for a Branch+PR repo, drop the *Landing under `Branch + PR`* paragraph for a direct-to-`main` repo, and remove the now-dead "Under *<other mode>*" clauses in the branch-naming and what-counts-as-a-landing paragraphs. Capture where worktrees live — and if that's inside the repo (e.g. `.claude/worktrees/`, where `EnterWorktree` puts them), check it's gitignored and add it if not — plus what local setup (`.env`, dependencies, build caches) doesn't carry into a fresh one. On *no*, delete that whole subsection.

  **Neither answer touches the Changelog discipline section.** The entry file is written the same way on `main`, on a branch and in a worktree, so nothing in that section is mode-dependent and there is nothing to keep or trim there. Don't add a conditional to it.

  (This overrides only the *where* half of the agent's generic default, "branch first" — it must **not** override the *when* half, "commit only when asked." Keep the template's note that the mode chooses where commits go, not when, or the agent reads "direct to `main`" as license to commit proactively the moment a change is done.)

Write to `<repo-root>/CLAUDE.md`.

### 5. Write subtree `CLAUDE.md` files

For each subtree path from step 3 **whose `CLAUDE.md` doesn't exist yet** (existing ones were set aside in step 4 for the audit in step 6):

1. **Identify the stack** — language and major frameworks for this subtree. Look in `docs/ARCHITECTURE.md` for descriptions, and at the subtree's package manifest if there is one (`package.json`, `*.csproj`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`).

2. **Read `templates/CLAUDE-subtree.md`** in this skill's folder.

3. **Fill the easy slots directly from the project:**
   - **Required tools** — at minimum, `LSP` for the subtree's primary language. For UI subtrees, mandate browser tools (Playwright MCP) — but first confirm Playwright MCP is actually installed and connecting; if it's missing, or only available headed (e.g. bundled by a plugin), offer to install it headless per `references/playwright-mcp.md`. Run that availability check once per run, not per subtree. Quote the load instructions if the tool is deferred.
   - **Testing** — read the actual test framework from the package manifest (Vitest / Jest / xUnit / pytest / etc.) and pin it. Keep the "do not introduce a different test framework without updating the architecture doc" closing line — it's a forcing rule, not boilerplate.
   - **Verification workflow** — include the 4-step Playwright MCP pattern (already in the template's catch-all section) for UI subtrees; skip it for backend/library subtrees where the test suite is the verification.

4. **Fill "Required skills" via the dynamic suggestion flow:**

   a. **Read the opinionated catalog** at `references/subtree-skills.md` in this skill's folder. It lists catalog items (skills, plugins, tools) with "when to mandate" hints.

   b. **Inspect what's actually installed** in the current Claude Code environment:
      - Parse the `available skills` block in the system context (it's there as a system-reminder during a session). Plugin-namespaced entries follow `plugin-name:skill-name`.
      - Deferred tools are discoverable via `ToolSearch` queries.
      - Optionally inspect `~/.claude/plugins/` or `.claude/plugins/` for plugin manifests.

   c. **Build a candidate list per subtree** by combining:
      - Catalog items whose "When to mandate" matches the subtree's nature.
      - Installed items not in the catalog that look clearly relevant (e.g., a `*-design-system` skill for a frontend subtree).

   d. **Ask the user, once per subtree:**
      > For `<subtree-path>`, suggested: `A` (installed), `B` (installed), `C` (install required from plugin `<x>`). Pick which to include — reply `all`, `none`, or list a subset.

   e. **Write the user's chosen items** into the Required skills section of that subtree's CLAUDE.md. Zero items is a valid answer — leave the section empty if the user picks `none`. Do not invent skill names that aren't installed and aren't in the catalog.

If the subtree directory doesn't exist yet, create it.

### 6. Audit existing `CLAUDE.md` files (if any)

For every `CLAUDE.md` set aside in step 4 because it already existed, run an audit instead of overwriting. The point is to bring an older or hand-written file up to this skill's *current* recommendations without clobbering the project-specific content someone added.

**a. Audit each file against the checklist.** Read `references/audit-checklist.md` in this skill's folder — it's the rubric (what every root and subtree `CLAUDE.md` should contain, the *why* behind each item, and what an outdated version looks like). For each existing file, classify every rubric item as **OK / Missing / Outdated / Stale**. Satisfied items aren't findings — only surface gaps. For existing subtree files you'll need the same stack/test-framework facts step 5 gathers (read the manifest + architecture doc) so you can judge S2/S3 fairly.

**a2. Ask the one thing the files can't tell you: are worktrees used here?** The rubric's R6b can't be judged by reading — a root `CLAUDE.md` that says "Direct to `main`" and has no worktree subsection looks complete either way, and under the old three-mode framing picking a landing mode *closed* the question. So when the root file has a current R6 mode but no worktree guidance **in any form**, ask the user question (b) from step 4 before presenting findings: is work in this repo ever isolated in a git worktree? Only *yes* turns it into a Missing finding. **Don't ask at all** when the file already carries worktree guidance in the old three-mode form ("Direct to `main`, parallel work in worktrees") — that file has already answered yes, and its finding is R6b *Outdated* (restructure the mode into the subsection), not Missing. Ask this even when the rest of the file audits clean — a file with zero other findings is exactly the case this question exists to catch, and it is the one place the audit is allowed to ask something before the consolidated prompt in **c**.


**b. Present findings, grouped by file.** One scannable line per finding: the item, its classification, and a few words of why it matters — e.g.:

> **`CLAUDE.md` (root)** — 3 findings
> - *Outdated:* reviews after every non-trivial edit — current convention is one review point, at the commit, so the accumulated diff is read as one change instead of a pass per prompt.
> - *Outdated:* ends every turn nudging the user toward a separate review pass — current convention drops the nudge; the commit gate is the one place it asks.
> - *Outdated:* changelog section is on the retired single-file numbered scheme, and this repo has an archive file — offered as the one migration item.
>
> **`src/web/CLAUDE.md`** — 1 finding
> - *Missing:* no browser-driven verification workflow for a UI subtree.
>
> **`src/api/CLAUDE.md`** — OK, nothing to update.

If nothing is outdated anywhere, say so plainly and move on — don't invent work. The **a2** question is the exception: ask it even on a file that audits clean, since a clean-looking file is precisely how the missing worktree subsection presents.

**c. Ask which files to update.** One consolidated prompt offering the files that have findings — the user replies `all`, `none`, or a subset. Let them narrow within a file too ("just the changelog fix in root"). A file with no findings isn't offered.

One exception to narrowing: the **changelog-layout migration** spans R1, R3, whichever section carries the landing sequence (R6 for a Branch+PR file, R6b once a worktree subsection is in play), the backfill of every existing entry into per-entry files, and the digest distilled from that same pass — and half of it is worse than neither. R3 alone routes entries into `docs/changelog/` while R1 still requires reading a file nothing writes, so new entries are invisible at session start; the landing sequence left alone still folds a fragment into a file nothing appends to; and the layout without the distillation drops the history from required reading and leaves an empty digest in its place. Offer it as a single item, priced (see R3's migration in the checklist — the distillation is the expensive step and gets delegated), and if the user tries to split it, say why and take all or none.

**d. Apply surgically to the chosen files.** Follow the "Applying updates" rules in `references/audit-checklist.md`: add Missing sections (filled from the project, not raw placeholders), rewrite Outdated sections to the current intent while keeping surrounding project-specific wording, fix Stale items, and **never delete custom sections the template never had**. Show the diff and confirm before writing each file. Files the user didn't pick are left exactly as-is.

### 7. Seed `docs/CHANGELOG-DIGEST.md`

The changelog is a folder of per-entry files, `docs/changelog/YYYY-MM-DD-<slug>.md`. Nothing needs seeding for that. What does need seeding is the digest, because it is `@`-referenced in required reading from the first session — a dangling `@`-reference is worse than an empty file.

If `docs/CHANGELOG-DIGEST.md` doesn't exist, create it with this header:

```markdown
# Changelog digest

Curated milestones and decisions a future session must not re-litigate. One to three lines each, naming the entry's slug in `docs/changelog/`. This file stays in required reading, so keep it small: when it passes ~100 lines, tighten the older half rather than growing it.

Full history is one file per change in `docs/changelog/`, named `YYYY-MM-DD-<slug>.md`. Each is a bounded lead (1–5 lines, ~20 words per line) written as durable project memory — what is now true that wasn't before, plus the why in a clause when it isn't obvious, not a recap of the diff — over a detail section, owed whenever the change leaves something the diff can't carry (an alternative rejected, a constraint that still binds, a known limit, a gap the verification didn't cover), never restating the lead, ~40 lines at most. A landed entry file is never renamed, moved, or deleted: its path is its address.

```

That second paragraph is deliberate duplication. The full discipline lives in root CLAUDE.md (step 4 wrote it, or step 6 brought it current), but this is the file an agent opens to record a milestone and the file a human opens to find out how the changelog works, so the convention has to be legible from here without reading CLAUDE.md.

**Do not create `docs/changelog/`.** The first entry creates it. Git can't track an empty directory, and a `.gitkeep` in a folder that will be permanently populated is dead weight nobody ever deletes. (This is a *fresh scaffold* rule. The step-6 migration does create the folder — it has hundreds of backfilled entries to put in it.)

**Don't seed the digest into a repo the audit just classified OK on the single-file scheme** (R3's migration rubric in `references/audit-checklist.md`). That would leave a digest sitting beside a `docs/CHANGELOG.md` whose `CLAUDE.md` never mentions it — a file with no reader and no writer. A repo keeps one convention or the other, never both.

If `docs/CHANGELOG-DIGEST.md` already exists, leave it alone.

### 8. Offer git-describe app versioning (optional)

Ask first; skip entirely if the user declines. This only pays off for a **deployed application** with a build/CI pipeline — a library, a one-off script, or a repo with no build step doesn't need it.

**The scheme.** Version the app from git, not a hand-bumped number. One annotated tag (`v0.1.0`) plus `git describe --tags --long --always --dirty` (e.g. `v0.1.0-32-g98af7ae`) is the version. New releases are just new annotated tags — no code change. (`--long` keeps the `-g<sha>` suffix even when HEAD is exactly on a tag — without it an exact tag prints just `v0.1.0` and the build loses its commit SHA; `--always` degrades to a bare short hash when there's no tag at all; `--dirty` flags uncommitted local builds.)

**The mental model that makes it work.** A deployed artifact — a container, a published binary, a built frontend bundle — has no `.git` and no git binary. So the version is **resolved once at build time, inside the checkout, and baked into the artifact**. The running app never shells out to git; it only reads the baked-in value. This is exactly how the git-versioning build tools work (MinVer, setuptools_scm, Nerdbank.GitVersioning), and every recipe in the reference follows that same shape — always baked in, never resolved at runtime.

**The universal runtime contract.** The app reads its version from whatever the build baked in for its stack — a stamped assembly/manifest, a generated version module, or an env var set at build — and falls back to `"unknown"` only when the bake step never ran. `git describe` runs in exactly one place: the build step (always inside the checkout, in CI and locally alike). Never make the running app resolve git itself.

If the user opts in:

1. **First tag** — if the repo has no tags yet, create the initial annotated one: `git tag -a v0.1.0 -m "Initial release marker"`. `git describe` needs at least one tag (or `--always`) to produce anything.

2. **Wire the ecosystem idiom.** Detect the stack — per subtree if they differ — and read the matching section of `references/app-versioning.md` in this skill's folder. It has copy-pasteable build-time-inject + runtime-read recipes for Node.js (backend & frontend), .NET, Python, Go, Rust, JVM, and containers. Apply the idiomatic mechanism (.NET MinVer, Python setuptools_scm, Go `-ldflags`, frontend bundler `define`, …) rather than bolting a generic runtime `git describe` onto a stack that already has a first-class one.

3. **The CI gotcha that bites every stack** — CI checkouts default to a shallow clone with no tags, so `git describe --tags` silently degrades to a bare hash and builds lose their version. Whatever the pipeline, make the checkout fetch full history and tags (GitHub Actions: `fetch-depth: 0`; otherwise `git fetch --tags --unshallow`). Flag this in any build config you touch.

### 9. Offer to install the companion skills (optional)

The root CLAUDE.md this skill writes *prefers* two skills but never depends on them. Same reasoning as Playwright MCP (`references/playwright-mcp.md`): a mandate that names a tool is worth more when the tool is actually there, so check, and offer — don't assume, and don't install unasked.

**The repo name and the skill name differ** — the repos are prefixed `claude-code-`, the skills are not. Install directory and skill name are always the short form:

| Skill | Install as | What the scaffolding uses it for | Repo |
| --- | --- | --- | --- |
| `fix-code` | `~/.claude/skills/fix-code/` | The tight review option at the root CLAUDE.md's commit gate (`/fix-code --fix`, offered second after `/code-review high --fix`). Without it the agent falls back to the same review inline — correct, just less calibrated. | `https://github.com/IngvarKofoed/claude-code-fix-code` |
| `git` | `~/.claude/skills/git/` | Commit/push delegated to a cheap Haiku subagent, keeping the main session's context and cost down. | `https://github.com/IngvarKofoed/claude-code-git` |

**a. Detect what's already there.** A skill counts as installed if it appears in your own available-skills listing, or if `~/.claude/skills/fix-code/` / `~/.claude/skills/git/` exists — the **short** names, not the repo names. **If both are installed, skip this step in silence** — no question, no mention in the summary.

**b. Ask only about what's missing.** For each missing skill, ask one `AskUserQuestion` question — both in a single call when both are missing, one question when only one is. Options: **install** (user-level, `~/.claude/skills/`, available in every repo) or **skip** (the CLAUDE.md fallback covers it). Never install without an explicit pick.

**c. Install user-level, under the short name.** Always name the target directory explicitly, or git derives it from the URL and you get `~/.claude/skills/claude-code-fix-code/` — a folder whose name doesn't match the skill:

```bash
git clone https://github.com/IngvarKofoed/claude-code-fix-code ~/.claude/skills/fix-code
git clone https://github.com/IngvarKofoed/claude-code-git      ~/.claude/skills/git
```

`skill-manager:add` is the nicer route when that plugin is present, since it registers the skill for auto-update — but confirm afterwards that the installed folder is the short name, and rename it if not.

**d. Note the reload.** A freshly installed skill isn't callable in the current session until skills reload — tell the user to run `/reload-skills` or restart Claude Code. Until then the CLAUDE.md fallback is what will actually run, which is fine.

Install user-level only. Don't offer to clone either skill into the repo — a nested clone can't be committed as-is, and a personal skill inside a shared repo decides the whole team's toolchain.

If a clone fails (no network, no access to a private repo), say so in one line and move on. The scaffolding works without either skill; that's the whole point of the fallback.

### 10. Print next steps

Print a short summary listing:

- Files created (scaffolded fresh), files updated (audited in step 6 and brought current), and files left alone (existing, no findings or not picked) — with paths.
- Reminders:
  - **Subtree CLAUDE.md tools, test framework, and verification workflow** were filled in from the project (package manifest + architecture). Required skills reflect the choices you made per subtree in step 5 — anything you didn't pick was left out. Skim and adjust.
  - **If you installed or reconfigured Playwright MCP during this run** (`references/playwright-mcp.md`), restart Claude Code or reconnect via `/mcp` before relying on the UI verification workflow — the new `mcp__playwright__*` tools aren't loaded in the current session yet.
  - **Review happens once, at the commit — not after every edit.** Deliberately: a pass per prompt only ever sees its own turn's edits and costs more than it catches. When you ask the agent to commit and the working tree holds non-trivial changes, it asks first, with three options: **`/code-review high --fix`** (the deep pass — widest brief, broadest coverage, recommended for the accumulated diff this gate usually sees), **`/fix-code --fix`** (the tight pass — consequence-rated, only serious and unambiguous repairs), or **commit as-is**. The agent runs your choice itself. `/code-review` ships with the harness, so option 1 always works; when `fix-code` isn't installed, option 2 becomes the same review run inline (one careful read for a small diff; a subagent fan-out via the Agent tool, findings verified, for a big one), auto-fixing what it finds, reporting grouped by severity, and closing with whatever it couldn't fully verify. Nothing has to be installed for the gate to work, so it never goes dead.
  - **What the gate does *not* defer** — tests, build, and each subtree's own verification workflow still run per change. Only the review waits for the commit. The gate is skipped for trivial diffs and for a diff a full pass already covered, asked once per commit request, and checked before any delegated commit (`/git commit`) hands off.
  - The root CLAUDE.md also carries a **multi-agent workflow ("ultracode") cost-tiering** rule — tier subagent model + effort to the work, keeping review and hard-reasoning stages on the strong model (verifying concrete findings can drop to the mid model). Inert unless the agent runs the Workflow tool.
  - **The changelog's session-start read is two `@`-references plus an action.** `docs/CHANGELOG-DIGEST.md` loads automatically; the recent window — `ls docs/changelog | tail -40`, then the newest 10 entry files — is something the session has to run, because no `@`-reference can express "the newest N files of a folder". The template numbers it and adds a self-check, but it can still be skipped, and it fails quietly when it is. If you want it deterministic, a session-start hook is the way (`update-config`); hooks are out of this skill's scope, so it's yours to add if you want it.
  - For project-specific domain skills (a design system, security rules, naming conventions), use `/skill-creator`. Add the required-skill mandate to the relevant subtree CLAUDE.md once the skill exists.

Stop. Do not start building domain skills or expanding the docs — those are separate decisions for the user.

## Safety

- Never overwrite an existing `CLAUDE.md` blind. An existing file goes through the audit pass (step 6): updates are surgical and additive, shown as a diff, and applied only to files the user picks. Custom sections the template never had are never deleted.
- Never modify `CONCEPT.md` or `ARCHITECTURE.md`.
- Never modify `docs/CHANGELOG.md` or `docs/CHANGELOG-archive.md` if they already exist — with exactly one exception, and it is the most consequential thing this skill can do: the changelog-layout migration (step 6, R3's migration rubric) **splits every entry into `docs/changelog/` and then removes both files from the working tree**. Only inside a migration the user explicitly picked; only after the verification gate passes (entry count in == file count out, every number accounted for in a marker); never on a mismatch; and in the same commit as the backfill, so it reverts as one diff. Entry content is copied verbatim and is never renumbered, reordered, or rewritten. Nothing is lost — git holds the originals — but this is a conversion, not an edit, so show the plan and the counts before doing it.
- Never modify a landed file in `docs/changelog/`. Its path is its address; renaming or deleting one breaks every citation to it and produces no conflict to catch.

## Scope

**In scope:** writing CLAUDE.md scaffolding, auditing and surgically updating existing CLAUDE.md files against the current recommendations (step 6, rubric in `references/audit-checklist.md`), seeding `docs/CHANGELOG-DIGEST.md` (step 7), migrating a repo from the retired single-file changelog to the per-entry `docs/changelog/` folder when the audit finds evidence of pressure (step 6 — one all-or-nothing item that splits every existing entry into a dated file, verifies the split, removes the original changelog files from the working tree, and distils the digest from the same pass; delegated as a fan-out, and only if the user picks it), optionally wiring git-describe app versioning (step 8, only if the user opts in — per-ecosystem recipes live in `references/app-versioning.md`), checking for the `fix-code` and `git` companion skills and offering to install the missing ones user-level (step 9, opt-in), and — for UI subtrees only — verifying Playwright MCP is installed and offering to install it headless if it isn't (`references/playwright-mcp.md`).

**Out of scope:** creating CONCEPT or ARCHITECTURE (the user does this), building domain skills (use `/skill-creator`), configuring hooks and general MCP-server or plugin management beyond the narrow Playwright-for-UI convenience above (use `update-config`), writing to user-side memory (per-user, not repo-side).
