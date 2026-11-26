# <Project name>

## Source of truth

These two docs are the source of truth for what we're building and how it's built. They run ahead of the code: much of what they describe isn't built yet, and building it is the work. Where existing code does something differently from the docs, don't change either one to match the other: point out the mismatch and ask which is right.

- @docs/CONCEPT.md — <a few words: what the project does>.
- @docs/ARCHITECTURE.md — <a few words: how it's built>.

## Changelog

Every change gets its own file in `docs/changelog/`, named `YYYY-MM-DD-<slug>.md` and written as part of the same change. There is no `CHANGELOG.md`, no entry numbers and no digest file.

- **Before you touch an area,** run `ls docs/changelog` and `grep -rli <term> docs/changelog/` with the words you'd use for it. That's how earlier decisions are found.
- **The slug names the thing, not the activity:** `expired-refresh-tokens`, not `auth-fixes`.
- **The lead**, 1–5 short lines under the `#` heading, says what is now true that wasn't before, and why when that isn't obvious. It doesn't recap the diff.
- **`## Detail`** holds what would be lost when the session ends: rejected alternatives, constraints that still bind, known gaps, what verification didn't cover. Name them in words someone would grep for, and don't restate the lead. Leave it out when there's nothing.
- **Never rename, move or delete an entry:** other docs cite it by slug. When a change revisits an earlier decision, cite that entry's slug.

Bad, a recap of the diff:

```
# Auth changes

Updated auth files, reworked middleware, added tests, renamed AuthHelper.
```

Good, `docs/changelog/2026-03-14-expired-refresh-tokens.md`:

```
# Expired refresh tokens rejected before session lookup

Auth now rejects expired refresh tokens before session lookup; stale sessions can
no longer silently renew. Validated at the middleware boundary, so handlers can
assume every request they see is current.

## Detail
- Rejected checking this in the session store: the store can't tell "expired" from
  "never existed", so the error the client got would have been wrong.
```

## Commits

<Of the first two bullets, keep the one for the mode the user picked and delete the other, then delete this note.>

- Commit straight to `main`, without a branch. <Add "Push after each commit." only if the user wants that; otherwise leave pushing out.>
- For each change, branch from `main` as `<naming, e.g. feat/<slug>>` and open a PR. <Who merges; whether `main` is protected.>
- When the user asks to commit a non-trivial diff, first ask with `AskUserQuestion`: run `/code-review high --fix` (recommended), or commit now. Skip the question for a trivial diff.
