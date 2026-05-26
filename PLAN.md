# opencode-skill — Plan

## Concept

Delegate coding tasks from **Claude Code** to **opencode CLI**.
Same pattern as `vibe-skill` (→ Mistral Vibe) and `gemini-skill` (→ Gemini CLI).

**Claude orchestrates. opencode codes. You review the diff.**

## Architecture

```
User in Claude Code: "/opencode add a login form"
  └─ SKILL.md loads
       └─ Claude decomposes task, writes self-contained prompt
            └─ ~/tools/opencode-delegate <workdir> <prompt> <timeout> <model>
                 ├─ writes prompt to temp file
                 ├─ runs: opencode run <prompt> --format json
                 │   --dangerously-skip-permissions --model <model> --dir <workdir>
                 ├─ Python parser streams: [read], [edit], [write], [search], [shell], [opencode]
                 ├─ aggregates token/cost from step_finish events
                 ├─ runs syntax checks on changed files
                 ├─ prints git diff --stat
                 └─ logs to ~/.local/share/delegate-runs.jsonl
```

## Files to create

| # | File | Purpose | Est. Lines |
|---|------|---------|------------|
| 1 | `tools/opencode-delegate` | Delegate script: bash wrapper + Python JSON parser | ~300 |
| 2 | `SKILL.md` | `/opencode <instruction>` main skill | ~180 |
| 3 | `OPENCODEON.md` | `/opencodeon` — enable auto-mode | ~10 |
| 4 | `OPENCODEOFF.md` | `/opencodeoff` — disable auto-mode | ~10 |
| 5 | `OPENCODESTATUS.md` | `/opencodestatus` — report status | ~20 |
| 6 | `README.md` | Install/usage docs | ~100 |

## Installation

```bash
git clone <this-repo> && cd opencode-skill
mkdir -p ~/tools ~/.claude/skills/opencode
ln -sf "$(pwd)/tools/opencode-delegate" ~/tools/opencode-delegate
chmod +x ~/tools/opencode-delegate
ln -sf "$(pwd)/SKILL.md" ~/.claude/skills/opencode/SKILL.md
# + auto-mode section in ~/.claude/CLAUDE.md
```

## `tools/opencode-delegate` design

### Shell wrapper
- Args: `workdir`, `prompt`, `timeout-secs` (default 300), `model` (default `opencode/deepseek-v4-flash-free`)
- Prompt → temp file (shell injection safety)
- Track git HEAD + file list before run
- Launch: `timeout $TIMEOUT opencode run $PROMPT --format json --dangerously-skip-permissions --model $MODEL --dir $WORKDIR`
- Capture exit code, check timeout (124)
- Post-run: syntax checks (py_compile, node --check, json.parse, bash -n)
- Print: `git diff --stat`
- Append JSON entry to `~/.local/share/delegate-runs.jsonl`

### Python stream parser
Handles opencode `--format json` event types:

| JSON event type | Map to | Fields used |
|---|---|---|
| `step_start` | `[step]` | — |
| `tool_use`, tool=`read` | `[read] <file>` | `part.state.input.filePath` |
| `tool_use`, tool=`edit` | `[edit] <file>` | `part.state.input.filePath`, `part.state.metadata.filediff` |
| `tool_use`, tool=`write` | `[write] <file>` | `part.state.input.filePath` |
| `tool_use`, tool=`grep`/`search` | `[search] <pattern>` | `part.state.input.pattern` |
| `tool_use`, tool=`bash`/`shell` | `[shell] <cmd>` | `part.state.input.command` |
| `tool_use`, tool=`glob` | `[tool] <pattern>` | `part.state.input.pattern` |
| `tool_use`, tool=other | `[tool] <name>` | `part.tool` |
| `text` | `[opencode] <text>` | `part.text` |
| `step_finish` | aggregate tokens+cost | `part.tokens.{total,input,output,reasoning}`, `part.cost` |
| `error` | `[ERROR] <message>` | `error.data.message` |

Aggregates per-step tokens into run totals.

## SKILL.md outline

1. **Frontmatter** — name, description, allowed-tools (bash, read_file, grep)
2. **Sub-commands** — `/opencodeon`, `/opencodeoff`, `/opencodestatus`
3. **Known limits** — no max-turns, free model delay, dangerously-skip-permissions required
4. **Step 1** — Detect workdir (git rev-parse)
5. **Step 2** — Decompose task (trivial/simple/medium/complex table)
6. **Step 3** — Write prompt (self-contained, grep-verify, atomic)
7. **Step 4** — Launch opencode-delegate
8. **Step 5** — Supervise (red flags table)
9. **Step 6** — Iteration (max 3 attempts, manual completion)
10. **Step 7** — Report to user

## Differences vs vibe-skill / gemini-skill

| Aspect | Vibe | Gemini | OpenCode |
|---|---|---|---|
| Needs pseudo-TTY | Yes (`script -q -c`) | No | No |
| Max turns | `--max-turns 12` | None (timeout) | None (timeout) |
| Auto-approve | `--trust` | `--yolo` | `--dangerously-skip-permissions` |
| Output format | Vibe streaming JSON | `stream-json` | `--format json` |
| Model override | `VIBE_ACTIVE_MODEL` | Hardcoded in script | `--model` flag |
| Per-step tokens | From session log | From result event | From `step_finish` events |
| Token calc | TOML pricing lookup | Hardcoded rates | From cost field in events |

## Shared log format

All delegates write to `~/.local/share/delegate-runs.jsonl` with field `delegate: "opencode"`.
Existing `~/tools/delegate-report` can query all runs regardless of delegate.

## Auto-mode

Flag file: `~/.local/share/opencode-auto.flag`
Toggle: `/opencodeon` (touch) → `/opencodeoff` (rm)

In `~/.claude/CLAUDE.md`:
```markdown
# opencode auto-mode
At the start of every user request that involves writing, editing, or fixing code:
1. Run `test -f ~/.local/share/opencode-auto.flag` (silent, no output to user).
2. If the flag exists → automatically invoke the `opencode` skill exactly as if the user had typed `/opencode <their full instruction>`. Do NOT ask first, do NOT explain — just delegate.
3. If the flag is absent → proceed normally.

The flag is toggled by `/opencodeon` and `/opencodeoff`.
```
