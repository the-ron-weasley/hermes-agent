---
title: "Antigravity Cli — Run agy with verified JSON streaming and headless workflows"
sidebar_label: "Antigravity Cli"
description: "Run agy with verified JSON streaming and headless workflows"
---

{/* This page is auto-generated from the skill's SKILL.md by website/scripts/generate-skill-docs.py. Edit the source SKILL.md, not this page. */}

# Antigravity Cli

Run agy with verified JSON streaming and headless workflows.

## Skill metadata

| | |
|---|---|
| Source | Optional — install with `hermes skills install official/autonomous-ai-agents/antigravity-cli` |
| Path | `optional-skills/autonomous-ai-agents/antigravity-cli` |
| Version | `0.3.0` |
| Author | Tony Simons (asimons81), Hermes Agent |
| License | MIT |
| Platforms | linux, macos, windows |
| Tags | `Coding-Agent`, `Antigravity`, `CLI`, `Auth`, `Plugins`, `Sandbox` |
| Related skills | [`grok`](../../optional/autonomous-ai-agents/autonomous-ai-agents-grok.md), [`codex`](../../bundled/autonomous-ai-agents/autonomous-ai-agents-codex.md), [`claude-code`](../../bundled/autonomous-ai-agents/autonomous-ai-agents-claude-code.md), [`hermes-agent`](../../bundled/autonomous-ai-agents/autonomous-ai-agents-hermes-agent.md) |

## Reference: full SKILL.md

:::info
The following is the complete skill definition that Hermes loads when this skill is triggered. This is what the agent sees as instructions when the skill is active.
:::

# Antigravity CLI (`agy`)

Operator guide for the Antigravity CLI, invoked as `agy`. Run all `agy`
commands through the Hermes `terminal` tool; inspect its config and logs with
`read_file`. This skill is reference + procedure — it does not wrap a network
API, so there is nothing to authenticate from Hermes itself.

## When to Use

- Installing, updating, or smoke-testing the `agy` binary
- Driving non-interactive `agy --print` / `agy -p` one-shots
- Debugging Antigravity auth, sandbox, permissions, or plugin state
- Reading Antigravity settings, keybindings, conversations, or logs

## Source and compatibility checks

Official headless reference: https://antigravity.google/docs/cli/headless

Use `agy --version`, `agy --help`, and `agy models` before building a command.
Verified with CLI 1.4.7, which supports JSON output, NDJSON streaming, stdin multi-turn streams,
`--json-schema`, `--effort`, and `--mode`. Recheck help after CLI upgrades.
A hub `up_to_date` result only compares the registered source; it does not prove
that the skill's contents match current CLI features. Do not upgrade the CLI,
change persistent settings, or disable permissions without operator approval.

## Mental model

Antigravity has two layers — keep them distinct or the guidance will be wrong:

1. **Shell wrapper commands** — `agy help`, `agy install`, `agy plugin`,
   `agy update`, `agy changelog`. Run these through the `terminal` tool.
2. **Interactive in-session slash commands** — `/config`, `/permissions`,
   `/skills`, `/agents`, etc. These only exist inside a running `agy` TUI
   session, not on the shell wrapper.

`agy help` shows the shell wrapper surface, NOT the in-session slash commands.

## Prerequisites

- The `agy` binary on PATH. Verify through the `terminal` tool:
  `command -v agy && agy --version`.
- No env vars or API keys required by this skill — Antigravity manages its own
  auth via the OS keyring / browser sign-in (see Authentication below).

## How to Run

Invoke every `agy` command through the `terminal` tool. Examples:

```
terminal(command="agy --version")
terminal(command="agy help")
terminal(command="agy plugin list")
terminal(command="agy --print 'Summarize the repo in 3 bullets'", workdir="/path/to/project")
```

For an interactive multi-turn TUI session, launch `agy` with `pty=true` (and
tmux for capture/monitoring), the same pattern the `codex` / `claude-code`
skills use. For one-shot smoke tests and scripted prompts, prefer
`agy --print` (non-interactive).

To inspect Antigravity's own files, use `read_file` on the paths under Core
paths below — do not `cat` them through the terminal.

## Delegation patterns

`agy` is a coding-agent backend in the same family as `codex` / `claude-code`,
so the same delegation shapes apply. Use these when handing real work (features,
fixes, reviews, second opinions) to Antigravity rather than just smoke-testing.

### One-shot (preferred for scripted prompts and second opinions)

```
terminal(command="agy -p 'Review this diff for bugs and security issues. Do not edit files.' --model gemini-3.8-flash-low --output-format json --print-timeout 4m", workdir="/path/to/repo", timeout=300)
```

`-p` is non-interactive: it runs the prompt and exits. Pick the engine with
`--model` (run `agy models` and use an exact returned model ID; do not copy
stale model names). Add extra context roots with repeatable `--add-dir`.
For reviews, explicitly forbid edits and tool actions outside the approved scope;
a prompt or `--mode plan` is not a filesystem read-only security boundary.

### Long / bounded runs (tests, builds, multi-file changes)

Background it and get notified on completion, the same as the `codex` skill:

```
# Use only after the operator has authorized this implementation scope.
terminal(command="agy -p 'Implement the change described in TASK.md and run the tests' --output-format stream-json --print-timeout 20m", workdir="/path/to/repo", background=true, notify=true)
# Inspect/manage the returned session through process_manage (discover its schema).
# Permission requests may fail without an interactive terminal. Do not silently
# add --dangerously-skip-permissions; obtain explicit approval for that override.
```

### Interactive multi-turn (PTY + tmux)

For a conversational session, launch `agy -i` (or bare `agy`) under `pty=true`
with tmux for `capture-pane` / `send-keys`, exactly the pattern documented in
the `codex` / `claude-code` skills. Resume later with `--continue` / `-c` or a
specific `--conversation <id>`.

### Parallel instances (batch sub-issue / worktree fan-out)

Create one git worktree per task and launch an independent `agy -p` in each
(background), then collect results — same worktree fan-out the `codex` skill
uses for batch issue fixing. Bound concurrency to what the machine and your
review capacity can absorb.

### Output formats, streaming and time limits

| Flag | stdout | Choose for |
| --- | --- | --- |
| `--output-format text` | Plain response (default) | Human-readable one-shots |
| `--output-format json` | One final JSON envelope | Result, status and usage |
| `--output-format stream-json` | NDJSON events | Live progress and tool monitoring |

Diagnostics go to stderr; keep stderr separate from stdout when parsing JSON.
The JSON envelope contains `conversation_id`, `status`, `response`,
`duration_seconds`, `num_turns`, `usage`, and `error` on failure. Resume by
`--conversation <conversation_id>`, not an invented `session_id` field.
Check both process exit status and terminal result status; partial output or an
`init` event alone is not success. Do not assume monetary cost is reported.

```bash
agy -p 'Explain git rebase in one sentence. Do not use tools.' \
  --output-format stream-json --print-timeout 2m \
  | jq --unbuffered -j 'select(.event=="step_update") | .step_update.text_delta // empty'
```

Use shell `set -o pipefail` when piping so a successful parser does not mask a
failed CLI. Capture the full stream separately if the final status is needed.
Events are `init`, `step_update`, and `result`; the last carries `.result`
with the JSON envelope. Text fragments are `.step_update.text_delta`, not a
Claude-style event schema. Inspect `tool_info` / `subagent_info` for tool steps.
Short responses may arrive entirely in one `DONE` event; intermediate token
chunks are not guaranteed. Read `.result.usage` rather than summing snapshots.

`--json-schema` accepts a schema string, schema-file path, or primitive type;
read `structured_output` in the JSON envelope or the stream's final `.result`.
It constrains the final result, not each progress event.

`--input-format stream-json` requires `--output-format stream-json`. Send one
NDJSON user event per line on stdin, e.g.
`{"event":"user","message":{"content":"Reply with OK. Do not use tools."}}`.
Keep the process alive for multiple turns: one initial `init`, one terminal
`result` per turn; close stdin at completion. Only text content blocks are
supported. See `references/cli-docs.md` for a complete multi-turn example.

There is no documented `--max-turns`. In CLI 1.4.7, `--print-timeout` defaults
to `0` (wait until completion), NOT five minutes. Set an explicit bound such as
`--print-timeout 20m` and allow additional time in the outer foreground tool
`timeout`. Background jobs use their process lifecycle for monitoring.

### Orchestration boundary

Antigravity is a **worker execution backend or third-opinion reviewer** — an
execution detail owned by the agent/profile running a task, NOT a first-class
orchestration primitive. Do not put `agy` on a kanban board as its own card or
treat it as a coordination layer; route work through the normal task graph and
let the assigned worker choose `agy` (vs. codex/claude-code/direct tools) as its
method. Reach for it explicitly only when the user asks, when a worker is
configured to wrap it, or when you want a Gemini-family cross-check against
another agent's plan or diff.

## Core paths

- Binary / entrypoint: `agy`
- App data dir: `~/.gemini/antigravity-cli/`
- Settings file: `~/.gemini/antigravity-cli/settings.json`
- Keybindings file: `~/.gemini/antigravity-cli/keybindings.json`
- Logs: `~/.gemini/antigravity-cli/log/cli-*.log`
- Conversations: `~/.gemini/antigravity-cli/conversations/`
- Brain artifacts: `~/.gemini/antigravity-cli/brain/`
- History: `~/.gemini/antigravity-cli/history.jsonl`
- Plugin staging: `~/.gemini/antigravity-cli/plugins/<plugin_name>/`

## Quick Reference

### Wrapper commands
- `agy changelog`
- `agy help`
- `agy install`
- `agy plugin` / `agy plugins`
- `agy update`

### Useful flags
- `--add-dir`
- `--continue` / `-c`
- `--conversation`
- `--dangerously-skip-permissions`
- `--print` / `-p`
- `--print-timeout`
- `--prompt`
- `--prompt-interactive` / `-i`
- `--sandbox`
- `--log-file`
- `--output-format text|json|stream-json`
- `--input-format text|stream-json`
- `--json-schema`
- `--disable-slash-commands`
- `--effort low|medium|high|xhigh|max`
- `--mode accept-edits|auto|plan` (execution mode, not a read-only guarantee)
- `--agent`, `--project`, `--new-project`
- `--version`

### Plugin subcommands (`agy plugin --help`)
- `list`, `import [source]`, `install <target>`, `uninstall <name>`,
  `enable <name>`, `disable <name>`, `validate [path]`, `link <mp> <target>`,
  `help`

### Install flags (`agy install --help`)
- `--dir`, `--skip-aliases`, `--skip-path`

### In-session slash commands
- **Conversation control:** `/resume` (`/switch`), `/rewind` (`/undo`),
  `/rename <name>`, `/clear`, `/fork`, `/reset`, `/new`
- **Settings & tools:** `/config`, `/settings`, `/permissions`, `/model`,
  `/keybindings`, `/statusline`, `/tasks`, `/skills`, `/mcp`, `/open <path>`,
  `/usage`, `/logout`, `/agents`
- **Prompt helpers:** `@` path autocomplete, `esc esc` clears the prompt (when
  not streaming), `!` runs a terminal command directly, `?` opens help

## Settings and permissions

### Common settings keys (`settings.json`)
- `allowNonWorkspaceAccess`
- `colorScheme`
- `permissions.allow`
- `trustedWorkspaces`

### Permission modes
`request-review`, `always-proceed`, `strict`, `proceed-in-sandbox`.

### Sandbox behavior
- `enableTerminalSandbox` is a boolean in `settings.json`; default `false`.
- Launch-time overrides (`--sandbox`, `--dangerously-skip-permissions`) can
  supersede persistent settings for the current session.

## Authentication behavior

- The CLI tries the OS secure keyring first.
- With no saved session, it falls back to browser-based Google sign-in.
- Locally it opens the default browser; over SSH it prints an authorization URL
  and expects the auth code pasted back.
- `/logout` removes saved credentials.

## Plugins

- Plugins stage under `~/.gemini/antigravity-cli/plugins/<plugin_name>/`.
- They can bundle skills, agents, rules, MCP servers, and hooks.
- `agy plugin list` returning no imported plugins is a valid empty state.

## Pitfalls

- `agy help` shows wrapper commands, not interactive slash commands.
- `agy --version` is the safe non-interactive version check; `agy version` is
  interactive and can fail without a real TTY.
- First place to look for failures: `~/.gemini/antigravity-cli/log/cli-*.log`
  (read with `read_file`).
- Don't confuse persistent JSON settings with launch-time overrides.
- `~/.gemini/antigravity-cli/bin/agentapi` is a thin wrapper to `agy agentapi`.
- On WSL, token storage is file-based, so auth issues are usually local-file /
  session-state problems, not browser-only problems.
- Workspace identity can depend on launch directory and the `.antigravitycli`
  project marker.
- Plain text is only the default. Select `--output-format json` or
  `--output-format stream-json` explicitly for machine-readable output.
- Bound print runs explicitly with `--print-timeout`; the CLI 1.4.7 default
  is `0` (unbounded), not five minutes. `--max-turns` is not documented.
- Use `--disable-slash-commands` for literal scripted prompts that must not
  expand slash commands or skills. It does not disable agent tool access.
- Never pass secrets or sensitive documents in a smoke-test prompt. Run tests
  from an isolated scratch directory and request no tool use.
- An initialized stream is not proof of a completed task. Require the final
  result, successful status, and real artifact/test evidence for coding work.

## Verification

Confirm the install is real and usable, all through the `terminal` tool (read
files with `read_file`):

1. `terminal(command="command -v agy")`
2. `terminal(command="agy --version")`
3. `terminal(command="agy help")`
4. `terminal(command="agy plugin list")`
5. `read_file` on `~/.gemini/antigravity-cli/settings.json`
6. `read_file` on the latest `~/.gemini/antigravity-cli/log/cli-*.log`
7. If needed, `read_file` on `~/.gemini/antigravity-cli/keybindings.json`

## Support files

- `references/cli-docs.md` — source links, JSON/NDJSON parsing, multi-turn stdin,
  structured output, safe smoke tests, and supplementary CLI notes.
