# claude-code-herdr-plugin

> Unofficial · v1.2.0 · MIT · Not affiliated with the herdr project or Anthropic.

**Drive a Codex sub-agent from Claude Code through a herdr pane — stream one JSON verdict per state change via the Monitor tool, or get one verdict per blocking turn. No screen-scraping, no status polling.**

```
                ┌── start --no-wait ──▶ Codex (in a herdr pane)
Claude Code ────┤                              │
                └◀── watch: one JSON verdict ───┘   (question · plan · completed · auto-closed)
                     per state change, streamed
                     to the Monitor tool
```

---

## Install · Update · Uninstall

All commands run inside Claude Code's slash prompt.

### Install (first time)

```
/plugin marketplace add yigitkonur/claude-code-herdr-plugin
/plugin install claude-code-herdr-plugin
/reload-plugins
```

What each line does:

1. **`/plugin marketplace add yigitkonur/claude-code-herdr-plugin`** — registers this repo as a single-plugin marketplace (reads `.claude-plugin/marketplace.json` from `main`).
2. **`/plugin install claude-code-herdr-plugin`** — installs the plugin from that marketplace (reads `.claude-plugin/plugin.json`).
3. **`/reload-plugins`** — makes Claude Code pick up the new skill without restarting.

One skill auto-activates when you delegate to Codex:

- **`claude-to-codex`** — drives the sub-agent end-to-end (spawn, send, wait, analyze, end)

### Update to a newer release

```
/plugin marketplace update claude-code-herdr-plugin
/plugin install claude-code-herdr-plugin
/reload-plugins
```

`marketplace update` refreshes the catalog from `main`; the subsequent `install` is what actually pulls the new files into your local plugin tree. The version number in [`.claude-plugin/plugin.json`](./.claude-plugin/plugin.json) is what Claude Code compares against to decide whether anything changed.

> **Don't use `/plugin update <name>` for marketplace-sourced plugins.** In current Claude Code builds it can remove the marketplace instead of upgrading the plugin. Use the three-line update flow above.

### Uninstall

```
/plugin uninstall claude-code-herdr-plugin
/plugin marketplace remove claude-code-herdr-plugin
```

The first removes the installed plugin; the second forgets the marketplace registration so it does not reappear in `/plugin marketplace list`.

## Prerequisites

Check each is in place; install the missing ones before using the plugin.

| Need | Check | Get it |
|---|---|---|
| **herdr** server running | `herdr status` → `running` | [ogulcancelik/herdr](https://github.com/ogulcancelik/herdr) |
| **Codex CLI** installed | `codex --version` | per Codex install docs |
| **Python 3** (3.6+) | `python3 --version` | ships on macOS / Linux |
| You're **inside a herdr pane** | `echo $HERDR_PANE_ID` non-empty | start one with `herdr` |

No `pip` or `npm` dependencies — the herdr socket transport is a vendored, stdlib-only copy of [`herdr-python-client`](https://github.com/54rt1n/herdr-python-client) under `skills/claude-to-codex/scripts/herdr_client/` (Apache-2.0).

## Verify (~30 s)

In Claude Code, say:

> have codex print HELLO and stop.

Expected: Claude backgrounds a spawn, a new herdr pane appears unfocused alongside yours, Codex boots (~5 s), runs the task, and Claude reports back within ~30 s with `state: completed, reason: marker_verified`. The pane closes automatically.

If that doesn't happen, `herdr status` is the first thing to check.

---

## Use it

Just talk to Claude Code. The driver picks the right spawn mode based on your wording:

| You say | Driver picks |
|---|---|
| *"have codex refactor src/foo.py"* | `--in pane` (default; quick side-task) |
| *"have codex audit the whole codebase, I'll review later"* | `--in tab` (long-running, visually revisitable) |
| *"have codex try a risky migration in isolation"* | `--in space` (fresh workspace) |
| *"have codex implement X on its own branch"* | adds `--worktree` (git worktree on `codex/<slug>`) |
| *"run 5 codex agents in parallel reviewing each file"* | a fleet of `--in pane` spawns |

You never touch the CLI for normal use.

### CLI (if you want to drive it manually)

```bash
SKILL=${CLAUDE_PLUGIN_ROOT}/skills/claude-to-codex   # set when the plugin is active

# Spawn + wait + verdict in one call
python3 $SKILL/scripts/codex.py start \
  --slug refactor-foo --in pane \
  --task "Refactor src/foo.py to remove duplication; keep behavior identical." \
  --expect src/foo.py --timeout 600

# Read result.state, run result.next_action.command, repeat until completed, then:
python3 $SKILL/scripts/codex.py end --session <id>
```

Full surface:

```
codex.py start  --task "<p>" --slug <safe-name>
                [--in pane|tab|space]  [--worktree] [--keep] [--keep-worktree]
                [--plan | --no-plan] [--expect PATH]... [--cwd DIR]
                [--marker STR] [--timeout 600] [--no-wait]
codex.py watch  --session <id> [--expect PATH]... [--no-auto-approve] [--no-close] [--timeout 1800]
                # STREAM one JSON verdict per state change (JSONL) — arm with the Monitor tool
codex.py send   --session <id> --message "<p>" [--expect PATH]... [--no-wait] [--timeout 600]
codex.py reply  --session <id> (--text "…" | --choice N | --approve | --reject) [--no-wait] [--expect PATH]...
codex.py await  --session <id> [--expect PATH]... [--timeout 600]
codex.py status --session <id> [--expect PATH]...      # one-shot, no wait
codex.py end    --session <id>                          # teardown per mode + delete state
codex.py sessions                                       # list live, prune dead
```

`--slug` is required. Python injects the completion marker and the "ask me if unsure" discipline; you supply no other scaffolding. Plan mode auto-engages when the task mentions a plan (`--no-plan` to disable).

### Two ways to drive it

- **Event-driven (recommended):** `start --no-wait` returns a `result.monitor` block; arm the **Monitor tool** with its `watch` command. The watch streams one verdict per state change — a question, a plan (full text, for approve/revise), completion — and **self-closes the pane on verified success**. React with `reply --no-wait`; permission gates are auto-approved.
- **One-shot:** background a blocking `start`/`send`/`reply`/`await`; each prints one verdict and exits (the harness notifies you). Drive turn-by-turn.

---

## Spawn modes (`--in`)

All three are **no-focus-change** — the human's view never shifts.

| `--in` | Footprint | Label rule |
|---|---|---|
| `pane` (default) | New pane split off caller's tab (`agent.start --split right`) | Pane label = `<slug>` (via `pane.rename`) |
| `tab` | New unfocused tab in caller's workspace, Codex full-width | Tab label = `<caller-space>-<caller-tab>-<slug>` |
| `space` | Fresh unfocused workspace + tab | Workspace = `<caller-tab-name>`; inner tab = `<slug>` |

**`--worktree`** (env `CODEX_WORKTREE=1`) is orthogonal: materializes a git worktree at `<repo>/.worktrees/codex-<slug>` on a new `codex/<slug>` branch from `HEAD`, and the spawned pane uses that path as `cwd`. On `end`, the worktree is removed **only if** the branch is fully merged into the caller branch AND the working tree is clean (`git status --porcelain` empty); otherwise it is kept and the verdict reports `worktree: {kept, branch, path, ahead, dirty, reason}`. `--keep-worktree` forces keep.

**`--keep`** (env `CODEX_KEEP=1`) skips per-mode teardown of the outermost resource (pane / tab / workspace) on `end` — for when you want Codex's pane to outlive the orchestration session.

---

## The JSON verdict

Pure JSON on stdout, one stable envelope per command:

```jsonc
{ "ok": true, "schema_version": "v1", "command": "start", "session": "cdx-3f9a",
  "result": {
    "state":  "completed|awaiting_clarification|awaiting_approval|permission_gate|working|no_signal|exited",
    "reason": "marker_verified|marker_unverified|artifacts_present|reported_done|free_text_question|multiple_choice|plan_approval|permission_request|working|timeout|no_signal|pane_gone",
    "summary": "<=200 chars",
    "plan": "<full plan text — never truncated — when awaiting approval>",
    "questions": ["…"], "options": [{"key":"1","label":"…","recommended":true}],
    "marker_found": true, "artifacts": [{"path":"…","exists":true,"bytes":4561}],
    "transcript_tail": "<cleaned last message>",
    "next_action": {"intent":"answer|approve|choose|verify|wait|start|nothing","command":"…","why":"…"},
    "worktree": null
  }, "error": null }
```

**Exit codes:** `0` = valid verdict (any state — read `result.state`); `2` = usage error; `3` = herdr environment error (server down — `error.code: HERDR_DOWN`, retryable); `4` = session/pane not found or died; `5` = internal error.

## Plan mode

```bash
codex.py start --slug build-coffee --plan --task "Build a single-page site for a coffee shop."
```

Returns `awaiting_approval` / `plan_approval` with the **full, untruncated plan** in `result.plan` and the menu in `result.options`. Approve with `reply --approve`; reject with `reply --reject`; pick another path with `reply --choice N`. After approval, `await` rides the implementation through its work bursts to `completed`.

---

## Why this exists

Driving another AI agent from a script is deceptively hard. A terminal agent's `idle` status means *the turn ended* — but that could be "task finished," "I have a question," or "here's a plan, approve it?" — all indistinguishable by status alone. Plans scroll off the visible screen. Sends get eaten during startup. Menus render a beat after the status settles. Panes renumber when a sibling closes.

The plugin encodes the answers to all of that (each verified live against Codex) so the orchestrator doesn't have to:

| Hard problem | Handled by |
|---|---|
| `idle`/`done` ≠ "complete" (finished vs question vs plan-menu vs blocked widget) | the analyzer → `state` / `reason` / `next_action` |
| Task sent during Codex's MCP/TUI init is silently lost | wait for a genuinely ready, stable composer |
| Long plans scroll off the visible screen | read full scrollback (`agent.read --source recent`) |
| A narrow split mangles plans & option labels (`Yes, impleme…`) | spawn Codex in a fresh full-width tab when `--in tab/space` |
| Completion via a keyword alone is unreliable | completion marker **AND** artifact verification |
| Codex emits idle blips *between* work bursts | a resume-grace loop that re-reads the screen |
| Plan approval redraws the screen blank | confirm Codex re-entered `working` before settling |
| A pane's slot id shifts when another pane closes | sessions keyed on the stable `terminal_id` |
| Spawn shape doesn't match the task (quick side helper vs visible work vs isolated repo) | `--in pane\|tab\|space` per spawn, with worktree-aware cwd |

The result: a token-efficient, resilient interface that never gets stuck and always tells the orchestrator the next move.

## Beyond one Codex

`codex.py` is the perfected path for a **single Codex session** — the focus of this plugin. For parallel fleets, other agents (Pi / Claude / OpenCode / Hermes), or custom tooling, drop to **raw herdr**: the deep-dives under `skills/claude-to-codex/references/` document the full substrate — the agent-vs-pane namespace, the send-keys vocabulary, the status model, events / subscribe, pane lifecycle, and the complete CLI + hidden IPC.

The `claude-to-X subagent` naming convention (the main skill is `claude-to-codex`) leaves room for sibling skills (`claude-to-pi`, `claude-to-opencode`, …) when comparable per-agent machinery exists.

---

## Migrating from the old skill-only install

Previous versions of this repo shipped as a raw skill (`git clone … ~/.claude/skills/skill-herdr`). v1.0.0 is a clean plugin cut — to migrate:

```bash
rm -rf ~/.claude/skills/skill-herdr
```

Then run the two `/plugin` commands above. The Python tool, session-state location (`~/.cache/skill-herdr/sessions/`), and verdict envelope schema are unchanged — only the install path and the skill name (`skill-herdr` → `claude-to-codex`) moved.

---

## Repository layout

```
.claude-plugin/
  plugin.json                # v1.1.0 plugin manifest
  marketplace.json           # single-plugin marketplace pointing at ./
skills/
  claude-to-codex/           # the only skill — drives Codex from Claude Code
    SKILL.md                 # the skill contract Claude Code loads (start here)
    scripts/
      codex.py               # the single Codex interface (agent-facing)
      _core.py               # shared engine: registry, spawn/send/wait, analyzer
      name_herdr_tab.py      # internal: slug validation + deterministic labels
      test_analyze.py        # deterministic regression test (no spawning)
      herdr_client/          # vendored herdr socket client (Apache-2.0)
    references/              # 15 single-topic deep-dives (load on demand)
LICENSE                      # MIT (top level)
README.md                    # this file
```

## Testing

The analyzer (the heart of the plugin) has a deterministic, spawn-free regression test:

```bash
python3 skills/claude-to-codex/scripts/test_analyze.py
python3 -m py_compile skills/claude-to-codex/scripts/_core.py skills/claude-to-codex/scripts/codex.py
```

It covers every verdict branch plus the hard-won edge cases: numbered-questions-aren't-a-menu, plan-never-truncated, plan-not-ballooned-on-completion, the blocked multiple-choice widget, transcript-tail cleanliness (banner / prompt-echo / internal-skill noise stripped), rejecting a marker that a plan merely *documents* vs one actually *printed*, plus all three `--in` mode dispatchers, the no-focus invariant, and the worktree round-trip (merged → removed; unmerged → kept). 13 cases + 18 regression checks.

Every behavior in `skills/claude-to-codex/references/codex-and-agents.md` was confirmed by driving a live Codex through herdr.

---

## Credits

The herdr socket transport under `skills/claude-to-codex/scripts/herdr_client/` is a vendored copy of
[`herdr-python-client`](https://github.com/54rt1n/herdr-python-client) by **Martin Bukowski**,
used under the **Apache License 2.0** (see `skills/claude-to-codex/scripts/herdr_client/LICENSE` and `NOTICE`). The
library source is unmodified; `_core.py` builds the Codex orchestration on top of it.

## License

This plugin is MIT, except `skills/claude-to-codex/scripts/herdr_client/` which is Apache-2.0 (see its `LICENSE`/`NOTICE`).
