# <Project> — <subtree-name>

<One-sentence description of what lives in this subtree. Refer to `docs/ARCHITECTURE.md` for the broader context.>

Contents: <list of projects / folders / packages here>.

## Required tools

- **`<Tool name>`** — <why it's required for work in this subtree>. <Load instructions if deferred, e.g. via `ToolSearch select:<name>`.>
- **`<Skill name>`** — <when to invoke it during work here>. <If the skill is project-specific, mention where it lives.>

## Testing

<Pin the test framework. Example: "Unit and integration tests live in `<Project>.Tests` using xUnit + `WebApplicationFactory` for integration. Do not introduce a different test framework without updating the architecture doc.">

## <Any other subtree-scoped rules>

Common ones (add only those that apply):

- **Verification workflow** — for subtrees where the obvious test (`run tests`, `tsc --noEmit`) doesn't catch what matters. Number the steps. Example (frontend / UI):
  > 1. Start the dev server.
  > 2. Drive the changed feature in a real browser via Playwright MCP.
  > 3. Check console messages and network requests for errors.
  > 4. Only then report the change as complete.
- **Required skills** — chain to project-specific skills (`foss-design-system`, security review, etc.) the agent must invoke for certain work.
- **Deployment / infra specifics** — pinned versions, env vars, runbook pointers.

Keep imperative ("you must…", "always…") with a concrete forcing action.
