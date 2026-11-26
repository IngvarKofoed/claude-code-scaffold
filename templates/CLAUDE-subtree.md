# <Project> — <subtree-name>

<One sentence: what lives in this subtree. Name an entry point only where the folder doesn't make it obvious, e.g. "`hive.ts` ties the services together; `server.ts` starts it.">

## Required tools

- **`<Tool name>`** — <why it's required for work in this subtree>.
- **`<Skill name>`** — <when to invoke it during work here>. <If the skill is project-specific, mention where it lives.>

## Testing

<Pin the test framework, and give the commands as the subtree's own scripts from its manifest, not a bare tool call: a script carries its pre-steps (a generated file, an env var), a bare call skips them. Example: "Vitest. Tests sit next to the code they test, as `*.test.ts`. Run `npm test` and `npm run typecheck` here before you report a change complete. Do not introduce a different test framework without updating the architecture doc.">

## <Any other subtree-scoped rules>

Common ones (add only those that apply):

- **Verification workflow** — for subtrees where the obvious test (`npm test`, a type check) doesn't catch what matters. Number the steps. Example (frontend / UI):
  > 1. Start the app with its dev script.
  > 2. Drive the changed feature via Playwright MCP — headless by default, so rely on the page snapshot, not a visible window.
  > 3. Check console messages and network requests for errors.
  > 4. Only then report the change as complete.
- **Required skills** — chain to project-specific skills (`foss-design-system`, security review, etc.) the agent must invoke for certain work.
- **Deployment / infra specifics** — pinned versions, env vars, runbook pointers.

Write only what `docs/ARCHITECTURE.md` doesn't already say: tools, commands, test conventions, checks. Every session loads that doc, so a rule copied from it is a second copy that drifts. A mechanical check — a search for a forbidden pattern — belongs in a test or a lint script; the rule here names the command, not the steps.

State each rule plainly, with the concrete action it requires — e.g. **"A type change lands in all three packages at once."** — rather than "you must…" or "always…".
