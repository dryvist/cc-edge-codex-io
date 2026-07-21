# Codex CLI Pack For Edge

## Overview

This Cribl Edge pack collects telemetry from 2 file monitor sources for the
OpenAI Codex CLI and forwards it to a Cribl Stream worker group for indexing,
analysis, and search:

1. **Session transcripts** — `$CODEX_HOME/sessions/**/rollout-*.jsonl` —
   Full-fidelity rollout files: one JSONL file per run, containing turn
   context, response items (messages, reasoning, tool calls, tool outputs),
   and session metadata
2. **Prompt history** — `$CODEX_HOME/history.jsonl` — A single append-only
   file with one entry per submitted prompt (session ID, timestamp, raw text)

## Environment prerequisite: `CODEX_HOME`

Codex CLI defaults `CODEX_HOME` to `~/.codex`. **Set `CODEX_HOME`
explicitly** wherever Cribl Edge runs — do not assume the default resolves
the way you expect on every OS/user account. This mirrors a lesson from the
Claude Code pack: an unset home-directory env var silently broke file
monitoring on macOS. Always point this pack at a concrete path via an
environment variable, never a hardcoded literal.

```bash
export CODEX_HOME="$HOME/.codex"
```

## Architecture

```text
Codex CLI ──writes──> $CODEX_HOME/sessions/YYYY/MM/DD/rollout-<ts>-<uuid>.jsonl
    |                                   |
    |                        Cribl Edge (file monitor)
    |                                   |
    └──writes──> $CODEX_HOME/history.jsonl
                                        |
                             Cribl Edge (file monitor)
                                        |
                             Cribl HTTP --> Cribl Stream Worker Group
```

## Data Sources

### Codex CLI Session Transcripts

The richest data source. Each rollout JSONL file contains one event per
line, with a top-level `type` of:

- `session_meta` — session ID, cwd, CLI version, model provider
- `turn_context` — per-turn model, sandbox policy, approval policy, effort
- `event_msg` — turn lifecycle events (started, model context window,
  collaboration mode)
- `response_item` — the actual conversation content; `payload.type` is one
  of `message`, `reasoning`, `function_call` / `function_call_output` (e.g.
  `exec_command`), or `custom_tool_call` / `custom_tool_call_output` (e.g.
  `apply_patch`)
- `compacted` — context-compaction markers

Use this data for:

- **Audit and compliance** — Complete record of every AI interaction,
  including shell commands and file patches
- **Tool usage analysis** — What tools the agent invokes and how often
- **Productivity metrics** — Session duration, turn counts, tool call
  patterns

### Codex CLI Prompt History

Lighter-weight log recording only the raw prompt text submitted by the
user:

- **Session ID** — Links the prompt to its full rollout transcript
- **Timestamp** — Unix seconds
- **Text** — The exact prompt sent

Use this data for:

- **Prompt pattern analysis** — What users actually ask for, independent of
  tool-call noise
- **Cross-referencing** — Join against session transcripts on `session_id`

## Pipelines

- `codex_sessions` — stamps `sourcetype: codex:cli:session`, `index: codex`,
  and OpenTelemetry-AI `llm.*` semantic-convention fields on every session
  transcript event (`datatype` already arrives via input metadata)
- `codex_history` — stamps `sourcetype: codex:cli:history`, `index: codex`,
  and `llm.agent.name` on every history event

Neither pipeline is named `main`; both feed a `__group` output per this
pack family's validator rules.

## Installation

Import the `.crbl` release asset into Cribl Edge (Manage Fleets → Packs →
Import Pack), then configure the `codex-cli-sessions` and
`codex-cli-history` inputs' `path` if `CODEX_HOME` isn't set on the worker,
and route the pack's outputs to your Stream worker group.

## Usage

Point Cribl Edge at a fleet with `CODEX_HOME` exported (see above), enable
the pack's inputs, and events start flowing to the `codex_sessions` and
`codex_history` pipelines automatically.

## Development

```sh
make install       # Node deps + git hooks
make docker-up      # start a local Cribl container
make test           # run the fixture-driven pipeline tests
make validate       # build the .crbl and print the deep-validator command
```
