# Catalog: skills, plugins, and tools worth mandating per subtree

A short, opinionated catalog of Claude Code ecosystem items the `scaffold` skill draws from when filling the **Required skills** slot in subtree `CLAUDE.md` files. The catalog is one input — the skill also inspects what's actually installed in the user's Claude Code environment and presents the combined suggestions to the user, who picks per subtree.

"Item" here covers anything in the Claude Code ecosystem you'd reference in a prompt: a `/skills` entry, a plugin (e.g. `csharp-lsp`, `typescript-lsp`), an MCP tool, a deferred tool.

## What this catalog is — and isn't

- **Is:** an opinionated suggestion list. Things the maintainer of this skill thinks are commonly worth considering per subtree.
- **Isn't:** the truth about any given environment. What's actually installed varies; the skill cross-references at scaffold time.
- **Isn't:** an enumeration of everything in Claude Code. New skills/plugins appear constantly; this catalog is intentionally curated and small.

Two things explicitly *not* covered here, because the scaffold skill picks them from elsewhere:

- **Language servers from the package manifest** — picked by reading `package.json` / `*.csproj` / etc.
- **The `simplify` skill** — mandated globally from root CLAUDE.md, never per-subtree.

## How the skill uses the catalog

1. Read this file's entries.
2. Inspect the current Claude Code environment: parse the `available skills` list visible in the session, look at deferred tools, optionally inspect `~/.claude/plugins/` or `.claude/plugins/`.
3. Per subtree, build a candidate list:
   - **Catalog items whose "When to mandate" matches** the subtree's nature.
   - **Installed items not in the catalog** that look relevant (e.g., a project-specific design-system skill the catalog doesn't name).
4. Present candidates to the user; mark each as `installed` or `install required`.
5. The user picks per subtree. Zero items is a valid answer.

## Catalog entries

### `frontend-design`
- **What:** Anthropic-bundled skill. Produces distinctive, non-generic UI; avoids AI-template aesthetics.
- **When to mandate:** frontend / UI subtrees where craft and polish matter.

### `security-review`
- **What:** Anthropic-bundled skill. Reviews changes for security issues.
- **When to mandate:** subtrees touching auth, crypto, payments, secrets, sensitive-data handling.

### `claude-api`
- **What:** Anthropic-bundled skill. Guidance for projects building with the Anthropic SDK; includes prompt caching patterns.
- **When to mandate:** subtrees that import `anthropic` / `@anthropic-ai/sdk`.

### `csharp-lsp` (plugin)
- **What:** Plugin providing the C# language server (`csharp-ls`). The `LSP` tool uses it for C# symbol navigation, references, hover.
- **When to mandate:** any subtree with C# code. The subtree CLAUDE.md should reference `LSP`; this plugin is the installation behind it.

### `typescript-lsp` (plugin)
- **What:** Plugin providing the TypeScript language server. The `LSP` tool uses it for `.ts` / `.tsx` / `.js` / `.jsx` symbol nav.
- **When to mandate:** any subtree with TypeScript or JavaScript code.

### Project-specific design-system skill (e.g. `<project>-design-system`)
- **What:** Project's own design system encoded as a skill — tokens, components, validation rules, icons.
- **When to mandate:** frontend subtree, if the project has a design system. Build via `/skill-creator` if absent.

### Project-specific domain rules skill
- **What:** Custom skill encoding a domain-specific constraint (naming convention, file layout, business rule, idiom).
- **When to mandate:** wherever the constraint applies.

## When zero items is the right answer

Many subtrees genuinely don't need anything beyond the global `simplify` mandate:

- Library packages with thorough unit tests.
- Pure backend subtrees with no security-sensitive surface.
- Docs / config subtrees.

For these, leave the Required skills slot empty rather than padding with items that don't fit.
