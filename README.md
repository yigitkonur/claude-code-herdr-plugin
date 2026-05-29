# claude-code-herdr-plugin

> unofficial · v1.3.0 · mit · not affiliated with herdr or anthropic.

**hand a task to codex from inside claude code — and get back one clean json verdict per turn instead of screen-scraping a terminal.** claude spins codex up in a background herdr pane, watches it, and tells you exactly what happened: done, asking a question, waiting on a plan, or stuck. you never poll, never parse a tui, never lose your place.

```
                ┌── start ──▶ codex (background herdr pane)
claude code ────┤                       │
                └◀── one json verdict ──┘   done · question · plan · auto-closed
```

## what you actually get

just talk to claude code. it reads your intent, picks where codex runs, drives it to the end, and reports back. real examples:

- **"have codex refactor src/foo.py while i write the tests"** → codex works in a side pane, you keep going; claude pings you the moment it's `completed` — marker printed **and** the file verified on disk. no babysitting, no "is it done yet?".
- **"build a landing page, but plan it first"** → codex proposes a plan, claude streams you the full plan text, you say approve, it builds and **closes its own pane** on success. you reviewed the plan without ever leaving your chat.
- **"build X but ask me 2–3 questions first"** → codex's questions show up as events; you answer in one line each; it finishes. the back-and-forth just works.
- **"try this risky migration in isolation"** → codex gets its own workspace (or a fresh git worktree on `codex/<slug>`) — your working tree and your on-screen view stay untouched.
- **"run 5 codex agents reviewing each file"** → a fleet of background panes; results stream back as each one finishes.

your focus never moves — every spawn and teardown stays in the background.

## install

inside claude code's slash prompt:

```
/plugin marketplace add yigitkonur/claude-code-herdr-plugin
/plugin install claude-code-herdr-plugin
/reload-plugins
```

update later: `marketplace update` → `install` → `reload-plugins` (don't use `/plugin update <name>` — in current builds it can drop the marketplace). uninstall: `/plugin uninstall claude-code-herdr-plugin` then `/plugin marketplace remove claude-code-herdr-plugin`.

## you'll need

- **herdr** running (`herdr status` → running) — [ogulcancelik/herdr](https://github.com/ogulcancelik/herdr)
- **codex cli** (`codex --version`)
- **python 3** — zero pip/npm deps; the herdr socket client is vendored, stdlib-only
- to be **inside a herdr pane** (`$HERDR_PANE_ID` set)

quick check: tell claude *"have codex print HELLO and stop"* — a pane appears unfocused, codex runs, claude reports `completed` in ~30s, the pane closes itself.

## how it drives codex

it's one tool — `scripts/codex.py` — and claude uses it for you. two modes:

- **event-driven (default):** `start --no-wait` spawns, then claude arms claude code's **monitor tool** with `codex.py watch`. the watch streams one json line per state change (question · plan · completion), auto-approves codex's rare permission gates, and self-closes the pane on verified success.
- **one-shot:** background a single blocking verb (`start` / `reply` / `await`), read its one verdict, react. turn-by-turn.

either way you act on `result.next_action` — never a screen scrape. plan mode auto-engages when the task mentions a plan.

## why it doesn't get stuck

driving a terminal agent from a script is deceptively painful: a pane going `idle` just means "the turn ended" — which is "done", "i have a question", or "approve my plan?", all identical from the outside. the plugin nails every one of these (each verified live against codex):

- `idle` ≠ done → the analyzer classifies finished vs question vs plan-menu vs blocked widget
- a task sent during codex's startup gets eaten → it waits for a real, stable composer and verifies the send landed
- long plans scroll off-screen → reads full scrollback, never truncates the plan
- completion-by-keyword lies → needs a printed marker **and** `--expect` files on disk
- idle blips mid-build, late-painting menus, pane-slot renumbering → all ridden out (screen-stability wait, sessions keyed on the stable terminal-id)
- your view never shifts → every spawn/teardown is focus-neutral

net: a token-efficient interface that always knows the next move and cleans up after itself.

## more

- the contract claude loads → [`skills/claude-to-codex/SKILL.md`](./skills/claude-to-codex/SKILL.md)
- driving fleets / other agents (pi, claude, opencode, hermes) raw → `skills/claude-to-codex/references/`
- test the analyzer → `python3 skills/claude-to-codex/scripts/test_analyze.py`

## credits · license

the herdr socket transport under `scripts/herdr_client/` is a vendored, unmodified copy of [`herdr-python-client`](https://github.com/54rt1n/herdr-python-client) by **martin bukowski** (apache-2.0). everything else is mit.
