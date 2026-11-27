# <Project> — <subtree-name>

## Testing

<Leave the section out unless one of these applies:
- One line for each real resource the code uses (an external API, a database, a repo, the user's own data): tests never touch it, and the fake or temp copy that stands in, naming its file. Example: "Tests never send real email: they use the fake mailer in `test/fakes.ts`."
- A check the test script doesn't run, or one that must run from elsewhere to catch breakage outside this subtree. Example: "After changing a type here, run `npm run typecheck` at the repo root."
- A script to use because the bare tool skips a pre-step: `npm run typecheck`, not `tsc --noEmit`.
- A coverage rule the tests already follow. Example: "Every endpoint has a route test.">

## Verification workflow

<UI subtrees only. Open with: "For a change to what the page shows or does, before you report it complete:"
1. Start the app: the exact command, with the setting that points it at scratch data, and what it opens without that setting.
2. Drive the changed feature in a browser via Playwright MCP, relying on the page snapshot.
3. Check console messages and network requests for errors.>

## Rules

<Rules about this subtree's code that pass the test below, each stated plainly with the action it requires. Example: **"A new endpoint gets a route test in the same change."**>

---

Instructions to the writer, not output:

Test every line: would a session that has read `docs/ARCHITECTURE.md`, the manifest and the folder act differently without it? If not, cut it. So: no sentence describing the subtree, no framework names, no plain `npm test`, no "tests sit next to the code", no rule `docs/ARCHITECTURE.md` already states, no tool or skill list, no claim about the machine ("headless", "installed"). A rule a test already enforces is cut. A mechanical check no test enforces is left out and reported in the summary as a test worth adding.

State rules plainly, not as "you must…" or "always…". Leave out a section with no surviving line. A subtree with no surviving line gets no file.
