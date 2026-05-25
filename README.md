# herdr-claude-plugin

> **Unofficial.** Not affiliated with the herdr project or Anthropic. Installable Claude Code plugin (v1.0.0).

**A Claude Code plugin that lets Claude drive a `codex` sub-agent end-to-end through one tool — over the **herdr** terminal multiplexer.**

Claude Code starts a task, hands it to Codex running in a side pane, and gets back **one structured JSON verdict** telling it exactly what happened and what to do next. No screen-scraping, no status polling, no babysitting — the Python layer absorbs every sharp edge and Claude just reads `result.state` and runs `result.next_action.command`.

```
Claude Code ──run a codex.py verb in the background──▶  Codex (in a herdr pane)
     ▲                                                        │
     └────────  one JSON verdict (state + next_action)  ◀─────┘
```

---

## Why this exists

Driving another AI agent from a script is deceptively hard. A terminal agent's `idle` status means *the turn ended* — but that could be "task finished," "I have a question," or "here's a plan, approve it?" — all indistinguishable by status alone. Plans scroll off the visible screen. Sends get eaten during startup. Menus render a beat after the status settles. Panes renumber when a sibling closes.

This plugin encodes the answers to all of that (each one verified live against Codex) so the orchestrator doesn't have to:

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
| Spawn shape doesn't match the task (quick side helper vs visible work vs isolated repo) | `--in pane|tab|space` per spawn, with worktree-aware cwd |

The result: a token-efficient, resilient interface that **never gets stuck** and always tells the orchestrator the next move.

---

## Requirements

- **herdr** — the AI-aware terminal multiplexer ([ogulcancelik/herdr](https://github.com/ogulcancelik/herdr)). Its server must be running (`herdr status` shows `running`).
- **Codex CLI** — the agent being driven (verified against `v0.132.0`, gpt-5.x).
- **Python 3** — **no `pip` dependencies**. The herdr socket transport is a *vendored*, zero-dependency copy of [`herdr-python-client`](https://github.com/54rt1n/herdr-python-client) (Apache-2.0) under `skills/claude-to-codex/scripts/herdr_client/`, so the plugin is fully self-contained.
- **Claude Code** — the orchestrator that invokes the plugin.

> The plugin talks to herdr over its Unix socket (`~/.config/herdr/herdr.sock`). It also works for Pi / Claude / OpenCode / Hermes panes via the lower-level scripts, but `codex.py` is the perfected, first-class path for Codex.

---

## Install

The repo doubles as a single-plugin marketplace (`.claude-plugin/marketplace.json`). In Claude Code:

```
/plugin marketplace add yigitkonur/herdr-claude-to-codex
/plugin install herdr-claude-plugin
```

Claude Code auto-discovers the two skills inside the plugin (`claude-to-codex` and `name-herdr-tab`) and invokes them whenever you delegate to Codex ("have codex do X", "run this in the background", "continue this session", …).

### Migrating from the old skill-only install

Previous versions of this repo shipped as a raw skill (`git clone … ~/.claude/skills/skill-herdr`). v1.0.0 is a clean plugin cut — to migrate:

```bash
rm -rf ~/.claude/skills/skill-herdr
```

Then run the two `/plugin` commands above. The Python tool, session-state location (`~/.cache/skill-herdr/sessions/`), and verdict envelope schema are unchanged — only the install path and the skill name (`skill-herdr` → `claude-to-codex`) moved.

---

## Quick start

The whole interaction is three steps: **background a verb → read the verdict → do `next_action`.**

```bash
SKILL=${CLAUDE_PLUGIN_ROOT}/skills/claude-to-codex   # set by Claude Code when the plugin is active

# 1. Start a task (run in the BACKGROUND so Claude Code is notified on completion).
python3 $SKILL/scripts/codex.py start \
  --slug refactor-foo \
  --task "Refactor src/foo.py to remove duplication; keep behavior identical." \
  --expect src/foo.py --timeout 600

# 2. The verdict comes back as one JSON envelope. Read result.state, then:
#      completed              -> done; run `end`
#      awaiting_clarification -> reply --text "<answer>"   (questions in result.questions)
#      awaiting_approval      -> reply --approve            (plan in result.plan)
#      permission_gate        -> reply --approve / --reject
#      working (timeout)      -> await --session <id>
#      no_signal              -> status --session <id>

# 3. Clean up when finished.
python3 $SKILL/scripts/codex.py end --session <id>
```

In Claude Code you don't run these by hand — you let it drive. You just say *"have codex build the landing page"* and it backgrounds `start`, waits for the notification, and acts on the verdict.

---

## The `codex.py` interface

```
codex.py start  --task "<p>" --slug <safe-name>
                [--in pane|tab|space]  [--worktree] [--keep] [--keep-worktree]
                [--plan] [--expect PATH]... [--cwd DIR]
                [--marker STR] [--timeout 600] [--no-wait]
codex.py send   --session <id> --message "<p>" [--expect PATH]... [--timeout 600]
codex.py reply  --session <id> (--text "…" | --choice N | --approve | --reject) [--expect PATH]...
codex.py await  --session <id> [--expect PATH]... [--timeout 600]
codex.py status --session <id> [--expect PATH]...      # one-shot, no wait
codex.py end    --session <id>                          # teardown per mode + delete state
codex.py sessions                                       # list live, prune dead
```

`--slug` is required. Python injects the completion marker and the "ask me if unsure" discipline; you supply no other scaffolding.

`--in` picks the spawn target shape (all unfocused — no human attention switch):

| `--in` | Footprint | Label rule |
|---|---|---|
| `pane` (default) | New pane split off the caller's tab (`agent.start --split right`) | Pane label = `<slug>` (via `pane.rename`) |
| `tab` | New tab in caller's workspace, Codex full-width | Tab label = `<caller-space>-<caller-tab>-<slug>` |
| `space` | Fresh workspace + tab | Workspace = `<caller-tab-name>`; inner tab = `<slug>` |

`--worktree` (also `CODEX_WORKTREE=1`) is orthogonal: it materializes a git worktree at `<repo>/.worktrees/codex-<slug>` on a new `codex/<slug>` branch from `HEAD`, and the spawned pane uses that path as cwd. On `end`, the worktree is removed **only if** the branch is fully merged into the caller branch AND the working tree is clean (`git status --porcelain` empty). Otherwise the worktree is kept; the verdict's top-level `worktree` field reports `{kept, branch, path, ahead, dirty, reason}`. `--keep-worktree` forces keep.

`--keep` (env `CODEX_KEEP=1`) skips per-mode teardown of the outermost resource (pane / tab / workspace) on `end` — useful when you want Codex's pane to survive past the orchestration session.

### The JSON verdict

Pure JSON on stdout, one stable envelope:

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

### Exit codes

| Code | Meaning |
|---|---|
| `0` | Valid verdict produced (any `state` — read `result.state`) |
| `2` | Usage error |
| `3` | herdr environment error (server down / socket missing — `error.code: HERDR_DOWN`, retryable) |
| `4` | Session / pane not found or died |
| `5` | Internal error |

---

## Plan mode

```bash
codex.py start --slug build-coffee --plan --task "Build a single-page site for a coffee shop."
```

Returns `awaiting_approval` / `plan_approval` with the **full, untruncated plan** in `result.plan` and the menu in `result.options`. Approve with `reply --approve`, keep planning with `reply --reject`, or pick a path with `reply --choice N`. After approval, `reply --approve` rides the implementation through its work bursts to `completed`.

---

## Beyond one Codex

`codex.py` is the perfected path for a **single Codex session** — the focus of this plugin. For parallel fleets, other agents (Pi / Claude / OpenCode / Hermes), or custom tooling, drop to **raw herdr**: the deep-dives under `skills/claude-to-codex/references/` document the full substrate — the agent-vs-pane namespace, the send-keys vocabulary, the status model, events / subscribe, pane lifecycle, and the complete CLI + hidden IPC — so you can compose your own orchestration on top of it.

The "claude-to-X subagent" naming convention (the main skill is `claude-to-codex`) leaves room for sibling skills (`claude-to-pi`, `claude-to-opencode`, …) when comparable per-agent machinery exists.

---

## Repository layout

```
.claude-plugin/
  plugin.json                # v1.0.0 plugin manifest
  marketplace.json           # single-plugin marketplace pointing at ./
skills/
  claude-to-codex/           # main skill — drive Codex from Claude Code
    SKILL.md                 # the skill contract Claude Code loads (start here)
    scripts/
      codex.py               # the single Codex interface (agent-facing)
      _core.py               # shared engine: registry, spawn/send/wait, analyzer
      test_analyze.py        # deterministic regression test (no spawning)
      herdr_client/          # vendored herdr socket client (Apache-2.0)
    references/              # 15 single-topic deep-dives (load on demand)
  name-herdr-tab/            # utility skill — deterministic tab/pane/workspace labels
    SKILL.md
    scripts/name_herdr_tab.py
README.md                    # this file
```

---

## Testing

The analyzer (the heart of the plugin) has a deterministic, spawn-free regression test:

```bash
python3 skills/claude-to-codex/scripts/test_analyze.py
python3 -m py_compile skills/claude-to-codex/scripts/_core.py skills/claude-to-codex/scripts/codex.py
```

It covers every verdict branch plus the hard-won edge cases: numbered-questions-aren't-a-menu, plan-never-truncated, plan-not-ballooned-on-completion, the blocked multiple-choice widget, transcript-tail cleanliness (banner / prompt-echo / internal-skill noise stripped), rejecting a marker that a plan merely *documents* vs one actually *printed*, plus all three `--in` mode dispatchers, the no-focus invariant, and the worktree round-trip (merged → removed; unmerged → kept).

Every behavior in `skills/claude-to-codex/references/codex-and-agents.md` was confirmed by driving a live Codex through herdr.

---

## Credits

The herdr socket transport under `skills/claude-to-codex/scripts/herdr_client/` is a vendored copy of
[`herdr-python-client`](https://github.com/54rt1n/herdr-python-client) by **Martin Bukowski**,
used under the **Apache License 2.0** (see `skills/claude-to-codex/scripts/herdr_client/LICENSE` and `NOTICE`). The
library source is unmodified; `_core.py` builds the Codex orchestration on top of it.

## License

This plugin is MIT, except `skills/claude-to-codex/scripts/herdr_client/` which is Apache-2.0 (see its `LICENSE`/`NOTICE`).
