# <Project> — <subtree-name>

<One sentence: what lives in this subtree. Name an entry point only where the folder doesn't make it obvious, e.g. "`routes.ts` wires the endpoints; `main.ts` starts the server.">

## Testing

<Only what a session can't learn from the manifest: what tests must never touch (a real API, a real database, a real repo) and the fakes that stand in for it; a check that has to run from elsewhere to catch breakage outside this subtree; a command whose pre-steps a bare tool call skips, named as the subtree's own script. Leave out the framework's name and a plain `npm test`: the session reads those from the manifest. Leave the section out when there's nothing of this kind. Example: "Tests never send real email: they use the fake mailer in `test/fakes.ts`. After changing a type here, run `npm run typecheck` at the repo root.">

## <Any other subtree-scoped rules>

Common ones (add only those that apply):

- **Verification workflow** — for subtrees where the obvious test (`npm test`, a type check) doesn't catch what matters. Number the steps. Example (frontend / UI):
  > 1. Start the app with its dev script, on scratch data if it otherwise uses real data (e.g. an env var that points the database at a temp file).
  > 2. Drive the changed feature via Playwright MCP — headless by default, so rely on the page snapshot, not a visible window.
  > 3. Check console messages and network requests for errors.
  > 4. Only then report the change as complete.
- **Deployment / infra specifics** — pinned versions, env vars, runbook pointers.

Write only what `docs/ARCHITECTURE.md` doesn't already say: commands, test conventions, checks, and rules about this subtree's code. Every session loads that doc, so a rule copied from it is a second copy that drifts. No list of tools or skills: a session has every tool installed on the machine, and a skill runs on its own description. A mechanical check — a search for a forbidden pattern — belongs in a test or a lint script; the rule here names the command, not the steps.

State each rule plainly, with the concrete action it requires — e.g. **"A new endpoint gets a route test in the same change."** — rather than "you must…" or "always…".
