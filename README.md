# scaffold

A [Claude Code](https://docs.claude.com/en/docs/claude-code) skill that bootstraps a repository's `CLAUDE.md` scaffolding once `docs/CONCEPT.md` and `docs/ARCHITECTURE.md` exist. Writes a root `CLAUDE.md`, a scoped `CLAUDE.md` per major subtree, and seeds `docs/CHANGELOG.md`. Refuses to overwrite existing files — shows a diff and asks instead.

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

Both templates carry placeholders (`<like this>`) you fill in during the scaffold run.

The scaffold run also, optionally, offers to wire git-derived app versioning (`git describe` baked in at build time) — skipped unless you opt in.
