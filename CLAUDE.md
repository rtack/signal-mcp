# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Commands

```bash
uv sync --dev                              # Install dependencies
uv run pytest                              # Run all tests
uv run pytest -q                           # Quiet
uv run pytest --cov --cov-report=term-missing   # With coverage
```

Run a single test file:
```bash
uv run pytest tests/test_server.py
```

Run a single test by name:
```bash
uv run pytest -k "test_send_message"
```

## Architecture

`src/signal_mcp/`:

| File | Role |
|------|------|
| `models.py` | Data classes (Message, Contact, Group, …) |
| `config.py` | Account detection, paths, daemon PID |
| `client.py` | Async JSON-RPC client wrapping the `signal-cli` daemon |
| `store.py` | SQLite message store (FTS5 search) — kept independent of `client.py`, no circular imports |
| `server.py` | MCP server — tool definitions (`TOOLS` list) + `call_tool()` dispatch; the only file that imports from both `client.py` and `store.py` |
| `desktop.py` | Signal Desktop DB importer |
| `translation.py` | Claude-powered translation (CLI only) |
| `cli.py` | Click CLI entrypoint |

### Adding a New Tool

1. Add the method to `client.py` (calls `_rpc(...)`)
2. Add a `Tool(...)` entry to the `TOOLS` list in `server.py`
3. Add an `elif name == "..."` branch in `call_tool()` in `server.py`
4. If the tool doesn't need the daemon (read-only from store), add it to `_DAEMON_FREE`
5. Add tests in `tests/test_server.py` and `tests/test_client.py`
6. Update the tool table in `README.md`

## Testing

Tests are fully mocked — no `signal-cli` install or real Signal account required to run them. No external network calls in tests; mock at the HTTP layer with `respx`. Always write tests when adding new functionality, in the same commit as the change.

## Don't Ask, Just Proceed

Same carve-out as `google-mcp`: routine edits, git operations, and tests in this repo proceed without asking first — merge (MR) and PR are excluded, see below.

## Branching & terminology: MR vs PR

Same three-branch model as `google-mcp`, local to this checkout:

- **`main`** — normally a pure mirror of `upstream/main` (`googlarz/signal-mcp`); doc-only exceptions may land here directly when explicitly instructed.
- **`local-dev`** — the integration branch actually built and run as the live MCP server. All feature branches start here, not from `main`: `git checkout -b feat/<name> local-dev`.
- **`feat/<name>`** — one per feature/fix, pushed to `origin` (the fork, `rtack/signal-mcp`) once ready. Fast-forward only, no merge commits: rebase onto `local-dev`'s current tip (`git rebase local-dev`) before merging. **One commit per PR** by default — squash before opening; a multi-commit PR needs a real reason, not just "that's how the work happened to land."

Two distinct actions, two distinct terms:

- **MR** = merging a `feat/<name>` branch into **`local-dev`** (the local fork/integration branch). A plain `git merge`, not a GitHub PR. **Gate: live-verified only** (a real MCP tool call against a real Signal account/message — green tests are necessary but not sufficient). No code review required.
- **PR** = a GitHub pull request opened against **`googlarz/signal-mcp`** (the external upstream repo). **Gate: everything MR requires, plus**: `README.md`'s tool table updated, and code-reviewed (e.g. `/code-review` on the branch diff).

**Merging (MR) is approval-gated, not Claude-forbidden.** Claude may run `git merge` into `local-dev` — but only after: (1) the MR gate below is satisfied, (2) Claude has explicitly proposed *that specific merge* and stated the gate is satisfied, (3) explicit approval for *that specific proposal*. Never merge preemptively. Opening a PR follows the same propose-then-approve pattern, at the PR gate.

Before an **MR**, confirm, then propose and wait for approval:
- [ ] Tests written in the same commit, `uv run pytest` clean
- [ ] Live-verified against a real Signal account/message
- [ ] Explicit, in-the-moment approval for *that specific merge*

Before a **PR**, confirm all of the above, plus:
- [ ] `README.md`'s tool table updated if a tool was added/changed/removed
- [ ] Code-reviewed — findings addressed or explicitly accepted
- [ ] PR targets `googlarz/signal-mcp` (upstream), not the fork's own `main`, unless explicitly told otherwise
- [ ] Explicit, in-the-moment approval for opening

## MR/PR Gates Require Evidence, Not Just a Checked Box

A checklist item above is satisfied only when its evidence is pasted into that turn — not recalled from memory or asserted in passing.

- **Tests written in the same commit** — name the test file(s) and how many cases were added/changed.
- **`uv run pytest` clean** — paste the real summary line from the run, not a claim that it passed.
- **Live-verified** — state the exact tool call made and its result. "Deferred" is not evidence of anything except that this item is unmet.
- **Explicit, in-the-moment approval** — quote the user's literal words; a paraphrase or an inference from an earlier, related turn doesn't count.
- **(PR only) README table updated** — name the section/rows changed.
- **(PR only) Code-reviewed** — name the method used and state findings addressed vs. explicitly accepted.

## Sync With Upstream Before Starting Any Branch Work

Before creating a new `feat/`/`fix/` branch (or resuming an existing one), check whether `upstream/main` has moved:

```bash
git fetch upstream main
git log --oneline main..upstream/main   # anything printed means it moved
```

If it has: fast-forward local `main` to match (`git update-ref refs/heads/main refs/remotes/upstream/main` — `main` never carries local-only commits, so this is always safe), **push it to `origin`** (`git push origin main` — this step is easy to skip since it doesn't affect anything local, but leaves the fork's actual GitHub `main` stale otherwise), then rebase `local-dev` onto the updated `main` (`git rebase main` from `local-dev`), resolving any conflicts.

## Worktree Isolation Required for All Work

Same rule as `google-mcp`: before editing any file, use `EnterWorktree` to get an isolated copy of the current branch. Never assume the working tree is clean or matches what was last left — a concurrent session may have uncommitted changes sitting in the shared checkout.

**Caveat:** `EnterWorktree` only creates worktrees for the session's own launch repo, or a repo nested inside it — it cannot target this repo from a session rooted elsewhere (e.g. a `~/brain`-rooted session), since `~/dev/ai/signal-mcp` is a sibling directory, not nested under `~/brain`. A session genuinely rooted here (launched with this directory as its working directory) can and should use `EnterWorktree` normally. A session rooted elsewhere uses the sanctioned raw `git worktree add <repo-root>/.claude/worktrees/<name>` workaround instead — see "Worktrees Always Go Through EnterWorktree" in the global CLAUDE.md for the exact procedure. Never commit straight to the shared checkout regardless of which session is doing the work.

## Live-Verifying a `feat/` Branch Before an MR

Signal has no sandbox/test account concept — live verification means a real message through a real linked account. Prefer sending to/from your own "Note to Self" (`send_note_to_self`) or a dedicated test contact/group when the tool under test allows it, so verification doesn't spam real contacts. Round-trip any state change (e.g. send + immediately delete/unsend) where feasible, same principle as google-mcp's calendar live-verify pattern.

Act as your own MCP client, no session or harness config involved:

1. `uv run signal-mcp serve` inside the worktree (or run the built server directly).
2. `mcp` (the Python MCP SDK) is already a dependency, so a small throwaway script can `stdio_client` into a subprocess running the branch's own server, call the new tool(s) for real, and log the raw response. Delete the script afterward — it's a one-off verification tool, never committed.
