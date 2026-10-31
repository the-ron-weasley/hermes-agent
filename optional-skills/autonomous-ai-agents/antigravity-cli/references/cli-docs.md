# Antigravity CLI docs, condensed

## Sources and compatibility

Authoritative headless documentation:
https://antigravity.google/docs/cli/headless

Cross-check commands against `agy --version`, `agy --help`, and `agy models`.
The streaming recipes below were exercised with CLI 1.4.7. Other
configuration, plugin and interactive notes below are supplementary guidance;
verify current subcommand help before acting. Hub update status is not a
content-correctness audit.

## Headless output contract

`-p`, `--print`, and `--prompt` run one prompt and exit. Output formats:

| Flag | stdout |
| --- | --- |
| `--output-format text` | Plain response, default |
| `--output-format json` | Single final JSON envelope |
| `--output-format stream-json` | One JSON event per line (NDJSON) |

Diagnostics go to stderr. Do not merge stderr into a JSON parser's input.
Authenticate interactively before unattended use; without cached credentials,
non-TTY runs fail rather than starting an interactive sign-in. Do not guess
credentials or initiate auth/configuration changes as a smoke-test side effect.

Final envelope fields: `conversation_id`, `status`, `response`,
`duration_seconds`, `num_turns`, `usage`; `error` appears on failure. With
`--json-schema`, `structured_output` and `json_schema` are included. Usage
contains input/output/thinking/cache-read/total token counts, not a guaranteed
monetary cost. Require process exit 0 and `status == "SUCCESS"` for a successful
smoke test. For implementation tasks also verify artifacts and tests.

The stream uses these event and payload pairs:

| `.event` | Payload | Meaning |
| --- | --- | --- |
| `init` | `.init` | cwd, tools, permission_mode; overrides such as model/agent |
| `step_update` | `.step_update` | step transition, text delta or tool details |
| `result` | `.result` | Final envelope |

`step_update` includes `step_index`, `state` (`ACTIVE`/`DONE`), and
`step_type` (e.g. `user_input`, `agent_response`, `tool`, `checkpoint`).
Text is in `text_delta`; tool data in `tool_info` (`name`, `parameters`,
`output`, optional `error`); subagent data in `subagent_info`. Optional duration
and usage can appear. Short answers can arrive as a single `DONE` text delta.
Do not confuse this schema with Claude streaming events, or add the final full
response to already concatenated deltas. Use terminal `.result.usage` rather
than summing cumulative snapshots.

### Parsing recipes

Run pipelines in a shell supporting `pipefail`; otherwise the parser's exit
code can hide an agy failure. For correctness-sensitive automation capture the
raw stream and validate its terminal result too.

```bash
set -o pipefail
agy -p 'Explain rebase briefly. Do not use tools.' \
  --output-format stream-json --print-timeout 2m \
  | jq --unbuffered -j 'select(.event=="step_update") | .step_update.text_delta // empty'
```

Extract terminal usage with:
`jq 'select(.event=="result") | .result.usage'`.
For single-envelope output use `--output-format json`, then read `.response`,
`.status`, `.usage`, and `.conversation_id` directly (no `.result` wrapper).

### Structured output

```bash
agy -p 'Return an object with ok set to true. Do not use tools.' \
  --output-format json --print-timeout 2m \
  --json-schema '{"type":"object","properties":{"ok":{"type":"boolean"}},"required":["ok"],"additionalProperties":false}'
```

Read `.structured_output` from JSON or `.result.structured_output` from the
terminal streaming event. Prefer `structured_output` over parsing `response`:
the latter can contain Markdown fences or duplicated serialized text even when
the validated object is correct. `--json-schema` also accepts a JSON schema-file path
or primitive type name. It constrains the final result, not progress events.

### Continuous stdin conversation

Requires both `--input-format stream-json` and `--output-format stream-json`.
Send one JSON user event per line, not raw text. No `-p` is necessary:

```bash
set -o pipefail
printf '%s\n' \
  '{"event":"user","message":{"content":"Do not use tools. Reply with exactly AGY_OK."}}' \
  '{"event":"user","message":{"content":"Do not use tools. Repeat the exact token from your previous answer."}}' \
  | agy --input-format stream-json --output-format stream-json --print-timeout 2m
```

There is one initial `init` and one `result` per turn. Keep stdin open while
sending follow-ups and close it when finished. `message.content` accepts a
string or a list of `{"type":"text","text":"..."}` blocks. Other block types
are not supported. Across separate processes, prefer explicit
`--conversation <conversation_id>` over `--continue` when concurrent runs could
change which conversation is most recent.

### Timeouts, safe invocation and verification

CLI 1.4.7 defaults `--print-timeout` to `0` (wait until complete), not `5m`.
Always set an explicit bound for unattended work and a slightly larger outer
foreground timeout. There is no documented `--max-turns` option.

Use model IDs returned by `agy models`. Supported current flags also include
`--effort low|medium|high|xhigh|max`, `--mode accept-edits|auto|plan`, and
`--disable-slash-commands`. The latter prevents prompt expansion, not tool use.
A plan-mode prompt is not a filesystem read-only boundary.

Run harmless smoke tests sequentially from a new directory inside `$TMPDIR`,
not a project or a directory containing private documents. Use a minimal prompt
such as `Do not use tools. Reply with exactly AGY_OK.`, an explicit returned
model ID, `--disable-slash-commands`, and `--print-timeout 60s`. Do not add
`--dangerously-skip-permissions` to solve headless permission failures without
operator approval.

Verify JSON decoding, final success status, exact response and conversation ID.
For NDJSON check the initial `init`, the final `result`, and absence of tool
steps. For stdin send two turns and check two successful results with the same
conversation ID and retained context. For schema output check the actual
`structured_output` object. Store raw stdout/stderr separately in scratch;
never substitute documentation examples for observed execution results.

## Install
- macOS/Linux: `curl -fsSL https://antigravity.google/cli/install.sh | bash`
- Windows PowerShell: `irm https://antigravity.google/cli/install.ps1 | iex`
- Windows CMD: `curl -fsSL https://antigravity.google/cli/install.cmd -o install.cmd && install.cmd && del install.cmd`

## Authentication
- Tries secure keyring first.
- If no saved session exists, falls back to browser-based Google sign-in.
- Local machine: opens the default browser.
- SSH/remote: prints a secure authorization URL, then expects the auth code to be pasted back.
- `/logout` removes saved credentials.

## Config and files
- Settings: `~/.gemini/antigravity-cli/settings.json`
- Keybindings: `~/.gemini/antigravity-cli/keybindings.json`
- Plugins: `~/.gemini/antigravity-cli/plugins/<plugin_name>/`

## Useful slash commands
- `/config`, `/settings`
- `/permissions`
- `/resume` / `/switch`
- `/rewind` / `/undo`
- `/rename <name>`
- `/model`
- `/keybindings`
- `/statusline`
- `/tasks`
- `/skills`
- `/mcp`
- `/open <path>`
- `/usage`
- `/logout`
- `/agents`

## Prompt helpers
- `@` path autocomplete
- `esc esc` clears prompt when not streaming
- `!` runs a terminal command
- `?` opens help / slash command list

## Permissions and sandbox
- Permission modes: `request-review`, `always-proceed`, `strict`, `proceed-in-sandbox`
- Launch overrides: `--sandbox`, `--dangerously-skip-permissions`
- Sandbox setting: `enableTerminalSandbox` in `settings.json` (default `false`)

## Plugins
- Plugins can bundle skills, agents, rules, MCP servers, and hooks.
- They are staged locally and auto-discovered once installed.

## Subagents
- `/agents` opens the panel for active/completed subagents.
- Subagents can run in parallel and request approvals.

## Keybindings
- `~/.gemini/antigravity-cli/keybindings.json`
- Malformed JSON falls back to defaults for broken actions.
- Docs list default bindings for clear, submit, cancel, exit, suspend, editor, approval yes/no, navigation, clipboard, undo/redo, and newline insertion.
