# Subtree toolchain catalog

A short reference the `scaffold` skill uses when filling in subtree `CLAUDE.md` placeholders. Suggestions, not commandments — the user reviews and adjusts. Catalog stays compact on purpose: it covers the common cases, not every niche.

## How the skill should use this

1. Identify the subtree's language/stack from `docs/ARCHITECTURE.md` and from any package manifest in the subtree itself (`package.json`, `*.csproj`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle`).
2. Pick the matching ecosystem section below.
3. Fill the subtree CLAUDE.md placeholders from the entries — refining if the manifest reveals a test framework the catalog default wouldn't have guessed.
4. If the subtree clearly serves a UI (HTML, React, Vue, Svelte, etc.), add the verification workflow. If it's pure backend / library code, skip that section.

## TypeScript / JavaScript

- **LSP** — `LSP` tool, typescript-language-server. Load via `ToolSearch select:LSP`. Prefer over Grep for symbol nav.
- **Test framework** — Vitest for Vite projects; Jest for older Node. Playwright Test (`@playwright/test`) for committed end-to-end.
- **UI verification** — apply the 4-step Playwright MCP workflow.

## C# / .NET

- **LSP** — `LSP` tool, csharp-ls.
- **Test framework** — xUnit; pair with `WebApplicationFactory` for ASP.NET Core integration.
- **UI verification** — N/A (tests are the verification).

## Python

- **LSP** — `LSP` tool, Pyright or Pylance.
- **Test framework** — pytest.
- **UI verification** — apply the 4-step workflow if Django/Flask/FastAPI serves HTML; skip for pure API services (tests are the verification).

## Rust

- **LSP** — `LSP` tool, rust-analyzer.
- **Test framework** — built-in `cargo test`.
- **UI verification** — N/A unless the crate is a Wasm/Yew/Leptos UI.

## Go

- **LSP** — `LSP` tool, gopls.
- **Test framework** — built-in `go test`.
- **UI verification** — N/A unless the binary serves a UI.

## Java / Kotlin

- **LSP** — `LSP` tool, jdtls (Java) or kotlin-language-server.
- **Test framework** — JUnit 5; Spring Boot integration via `@SpringBootTest`.
- **UI verification** — N/A for typical backends.

## End-to-end test subtrees (any host language)

- **LSP** — for the spec language (usually TypeScript).
- **`@playwright/test`** (or the project's chosen e2e framework) — the committed test framework. Distinct from the Playwright MCP plugin.
- **Playwright MCP** — optional, useful while authoring a new spec; not used by committed tests.

## Common skills to mandate per subtree

Skills are project-specific — **do not invent skill names**. But these patterns recur. If the project has the relevant concern, surface a placeholder hint pointing at the right skill kind; if it has the actual skill already, mandate it.

- **Design system** — projects with their own design system (FOSS instruments, internal brand) usually build a `<project>-design-system` skill. Mandate from the frontend subtree's CLAUDE.md.
- **`frontend-design`** — Anthropic-bundled, produces distinctive (non-generic) UI. Mandate from frontend subtree if available.
- **`security-review`** — Anthropic-bundled. Mandate from any subtree touching auth, crypto, payments, or sensitive data handling.
- **`simplify`** — already mandated globally from root CLAUDE.md; **don't** re-mandate per subtree.

If the project doesn't have a matching domain skill yet, leave the "Required skills" placeholder visible so the user knows it's an open question. They can build the skill via `/skill-creator` and add the mandate later.

## Workflow patterns by subtree type

These cut across language and are worth surfacing when the subtree fits:

- **UI subtrees** — verification workflow (the 4-step Playwright MCP pattern is the canonical version).
- **Library subtrees** — usually no verification workflow; the test suite is the truth. Skip.
- **Service / backend subtrees** — verification typically = run integration tests against the service. Skip workflow unless the service has manual smoke-test steps.
- **Infra / deployment subtrees** — often have runbook-style verification (e.g., "after `terraform apply`, run the smoke-check script"). Number the steps if so.
- **Documentation subtrees** — no verification needed beyond review.
