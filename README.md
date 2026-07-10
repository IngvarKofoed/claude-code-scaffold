# scaffold

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill that bootstraps a repository's `CLAUDE.md` scaffolding once `docs/CONCEPT.md` and `docs/ARCHITECTURE.md` exist. Writes a root `CLAUDE.md`, a scoped `CLAUDE.md` per major subtree, and seeds `docs/CHANGELOG.md`. It never overwrites an existing `CLAUDE.md` blind: when a file already exists, it **audits** it against the skill's current recommendations, reports per file what's missing or outdated, and asks which files to update — applying surgical, additive edits that preserve your project-specific content.

The opinion behind it: anchoring docs + tight imperative `CLAUDE.md` files + numbered append-only changelog + the `code-review` skill mandate is the minimal scaffolding that makes a repo productive to work in with Claude. Domain skills, memory, and hooks are out of scope — add those separately when the project actually needs them.

## Add to a project

### As a git submodule (recommended)

Pinned to a specific commit, easy to update.

```bash
git submodule add git@github.com:IngvarKofoed/claude-code-scaffold.git .claude/skills/scaffold
git submodule update --init
```

### As a one-time clone

If you don't want a submodule tracked in your repo, just clone the folder directly.

```bash
git clone git@github.com:IngvarKofoed/claude-code-scaffold.git .claude/skills/scaffold
```

You can later delete the inner `.git/` if you want to vendor it in.

## Use

Once installed, the skill becomes available in your Claude Code session. Invoke it by:

- typing `/scaffold` directly, or
- asking in natural language: *"scaffold this repo"*, *"set up CLAUDE.md"*, *"I've written my CONCEPT and ARCHITECTURE, now what?"*

It will check that `docs/CONCEPT.md` and `docs/ARCHITECTURE.md` exist (refusing otherwise), look at their sizes, identify subtrees from `ARCHITECTURE.md`, then write the scaffolding.

### Auditing an existing repo

On a repo that already has `CLAUDE.md` files, the same skill audits them instead of overwriting. Trigger it with:

- *"audit my CLAUDE.md files"*, *"are my CLAUDE.md up to date?"*, *"check my CLAUDE.md against the latest scaffold conventions"*, or *"update CLAUDE.md to the current conventions"*.

It reads each existing `CLAUDE.md`, compares it to the rubric in `references/audit-checklist.md` (the read-first block, changelog discipline, the `code-review` mandate, the git workflow, per-subtree tools/testing/verification, …), lists per file what's **missing**, **outdated**, or **stale**, and asks which files to update. Updates are surgical — it adds or refreshes only the recommendation-driven sections and leaves your custom content untouched, showing a diff before writing. A run can scaffold the missing files and audit the existing ones at the same time.

## Update

```bash
cd .claude/skills/scaffold
git pull origin main
```

If installed as a submodule, also commit the bumped submodule pointer in the parent repo:

```bash
cd <repo-root>
git add .claude/skills/scaffold
git commit -m "Bump scaffold skill"
```

## What's in the skill

- `SKILL.md` — the workflow Claude follows.
- `templates/CLAUDE-root.md` — starter template for the repo-root `CLAUDE.md`.
- `templates/CLAUDE-subtree.md` — starter template for per-subtree `CLAUDE.md`.
- `references/subtree-skills.md` — opinionated catalog of skills/plugins/tools to mandate per subtree.
- `references/app-versioning.md` — per-ecosystem git-describe versioning recipes for the optional versioning step (Node.js, .NET, Python, Go, Rust, JVM, containers).
- `references/audit-checklist.md` — the rubric for auditing an existing `CLAUDE.md`: what each file should contain, why, what an outdated version looks like, and how to apply surgical updates.
- `references/playwright-mcp.md` — for UI subtrees: how to verify Playwright MCP is installed and offer to install it headless (own user-scope server, `--headless`) if it isn't.

Both templates carry placeholders (`<like this>`) you fill in during the scaffold run.

The scaffold run also, optionally, offers to wire git-derived app versioning (`git describe` baked in at build time) — skipped unless you opt in.
