# Playwright MCP: verify it's installed, offer to install it headless

The scaffold skill mandates **Playwright MCP** browser tools for UI subtrees (step 5.3 "Required tools", and the verification workflow in `templates/CLAUDE-subtree.md`). A `CLAUDE.md` that tells the agent to "drive the feature in a real browser via Playwright MCP" is a dead letter if Playwright MCP isn't actually installed. So before writing that mandate, confirm it's there — and if it isn't, offer to install it.

Run the check **once per scaffold run**, not once per UI subtree.

## Why headless is the default

`@playwright/mcp` (Microsoft's Playwright MCP server) is **headed by default** — it pops a visible browser window on every run. For agent-driven verification that's noise: it steals window focus, and it can't run on a headless CI box. Headless is the right default; a visible window is the exception you turn on to watch a run while debugging.

The single flag that controls it is `--headless` on the server's launch command.

## Detect what's already there

Three cases:

1. **Own MCP server** (preferred) — run `claude mcp get playwright`. A healthy install prints `Status: ✔ Connected` with `Args: @playwright/mcp@latest --headless`. If the args are missing `--headless`, it's installed but headed — offer to fix the flag.
2. **Bundled by a plugin** — e.g. the official `playwright` plugin. Check `claude plugin list` for a `playwright@…` entry. The plugin launches the server **headed, with no way to pass `--headless`** short of editing its cached `.mcp.json` (which gets wiped on plugin update). Its tools show up under `mcp__plugin_playwright_playwright__*`.
3. **Not installed** — neither of the above, and no `mcp__playwright__*` / `mcp__plugin_playwright_playwright__*` tools in the session.

## The recommended install: own server, user scope, headless

Configure Playwright MCP as a server the user owns, so the config survives plugin/marketplace updates and you control the flags:

```bash
claude mcp add playwright -s user -- npx @playwright/mcp@latest --headless
```

- `-s user` → available in every project, not just this repo.
- Verify with `claude mcp get playwright` → expect `Status: ✔ Connected`.

If a **plugin** is already providing a (headed) Playwright, offer to replace it with the owned headless server and drop the plugin, so there aren't two overlapping browser toolsets carrying duplicate tool cost:

```bash
claude plugin uninstall playwright@<marketplace>   # e.g. playwright@claude-plugins-official
```

Tool names change from `mcp__plugin_playwright_playwright__*` to `mcp__playwright__*` — transparent in use.

## Watch for a competing browser MCP

Installing Playwright isn't finished if a *different* browser MCP is still configured — a third-party `chrome-mcp`, a puppeteer server, a Chrome-DevTools MCP, etc. Two browser toolsets loaded at once means the agent may reach for the wrong one, plus duplicate tool cost in every session. If a UI subtree mandates Playwright, scan `claude mcp list` for another browser server and **flag the overlap to the user** — recommend removing the one they're not standardizing on so "use the browser" is unambiguous.

Two gotchas worth surfacing when you flag it:

- **The competing server may live in an unexpected `.mcp.json`.** `claude mcp remove <name> -s project` operates on the *current repo's* `.mcp.json`; if it reports "not found" even though `claude mcp list` shows the server, it's defined elsewhere — commonly a home-level `~/.mcp.json` that loads for every session under `~`. Run `claude mcp get <name>` to see the scope, and if the CLI can't reach it, edit the owning `.mcp.json` directly. A stale `enabledMcpjsonServers` entry in `.claude/settings.local.json` may also need clearing.
- **The built-in "Claude in Chrome" (`mcp__claude-in-chrome__*`) is not a configured MCP server** — it's Claude Code's Chrome-extension connector, so there's no `claude mcp remove` for it; it's managed at the extension level and stays inert unless explicitly invoked. Don't promise to remove it via MCP config.

Removing browser servers is the user's call — flag and recommend; don't treat it as automatic housekeeping (general MCP management belongs to `update-config`).

## Restart to pick it up

MCP config changes don't apply to the already-running session. After installing (or changing the flag), the user must **restart Claude Code**, or reconnect via `/mcp`. The current session still has the old tools loaded — say so in the next-steps summary so the user isn't surprised the new `mcp__playwright__*` tools aren't there yet.

## Toggling headed for debugging

To watch a run in a visible window temporarily, swap the flag back off:

```bash
claude mcp remove playwright -s user
claude mcp add playwright -s user -- npx @playwright/mcp@latest        # headed
```

Re-add with `--headless` when done.

## Browser binaries

`@playwright/mcp` drives real browser binaries. On a fresh machine the first run may need them downloaded once (`npx playwright install chromium`). If a run fails with a "browser not found" / "executable doesn't exist" error, that's the fix — not a config problem.
