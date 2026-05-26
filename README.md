# opencode-skill

![MIT License](https://img.shields.io/badge/license-MIT-blue.svg)

**Claude orchestrates. OpenCode codes. You review the diff.**

A Claude Code skill that delegates coding tasks to **OpenCode CLI** and supervises the result — the same pattern as [vibe-skill](https://github.com/pcx-wave/vibe-skill) and [gemini-skill](https://github.com/pcx-wave/gemini-skill).

---

## What it does

When you type `/opencode <instruction>` in Claude Code, this skill:

1. Decomposes the task into atomic sub-tasks (if needed)
2. Writes a self-contained prompt for OpenCode
3. Runs `opencode-delegate` — a script that launches OpenCode CLI in non-interactive mode
4. Streams structured output: `[read]`, `[edit]`, `[write]`, `[search]`, `[shell]`, `[opencode]`
5. Runs post-run syntax checks on modified `.py`, `.js`, `.json`, `.sh` files automatically
6. Reports the git diff and any issues to you
7. Appends a structured JSON entry to `~/.local/share/delegate-runs.jsonl` (tokens, cost, duration, exit code)

---

## Prerequisites

- [OpenCode CLI](https://opencode.ai) installed (`opencode --version`)
- [Claude Code](https://claude.ai/code) with skills enabled
- `python3` available (for the streaming parser and syntax checks)
- A git repository to work in

---

## Installation

```bash
git clone <your-repo-url> && cd opencode-skill
mkdir -p ~/tools ~/.claude/skills/opencode ~/.claude/skills/opencodeon ~/.claude/skills/opencodeoff ~/.claude/skills/opencodestatus
ln -sf "$(pwd)/tools/opencode-delegate" ~/tools/opencode-delegate
chmod +x ~/tools/opencode-delegate
ln -sf "$(pwd)/SKILL.md" ~/.claude/skills/opencode/SKILL.md
ln -sf "$(pwd)/OPENCODEON.md" ~/.claude/skills/opencodeon/SKILL.md
ln -sf "$(pwd)/OPENCODEOFF.md" ~/.claude/skills/opencodeoff/SKILL.md
ln -sf "$(pwd)/OPENCODESTATUS.md" ~/.claude/skills/opencodestatus/SKILL.md
```

### Verify the install

```bash
# Check opencode is available
opencode --version

# Test the delegate script
~/tools/opencode-delegate /tmp "Say hello in one sentence." 60
# Should print: [opencode] Hello! ...
```

---

## Usage

In a Claude Code session, just describe what you want:

```
/opencode add a dark mode toggle to the settings page
```

```
/opencode the login form is not validating the email field — fix it
```

Claude will decompose the task, write the OpenCode prompt, supervise execution, and report the diff.

---

## Auto-delegate mode

For frictionless delegation, enable auto-mode once in your Claude Code session:

```
/opencodeon      — every code request is automatically delegated to OpenCode
/opencodeoff     — return to normal Claude behaviour
/opencodestatus  — check if auto-mode is ON or OFF
```

With `opencodeon` active, just talk to Claude normally — coding tasks go to OpenCode automatically.

---

## How opencode-delegate works

```
Claude Code
  └─ /opencode <instruction>
       └─ SKILL.md logic
            └─ ~/tools/opencode-delegate <workdir> <prompt> [timeout] [model]
                 ├─ writes prompt to temp file (avoids shell injection)
                 ├─ runs: opencode run <prompt> --format json
                 │   --dangerously-skip-permissions --model <model> --dir <workdir>
                 │   └─ no pseudo-TTY needed (unlike Vibe)
                 ├─ pipes JSON events through Python parser
                 │   └─ handles: step_start / tool_use / text / step_finish / error
                 │   └─ prints [read] / [edit] / [write] / [search] / [shell] / [opencode]
                 ├─ aggregates real token counts from step_finish events
                 ├─ runs syntax checks on modified .py, .js, .json, .sh files
                 ├─ prints git diff --stat
                 └─ appends JSON entry to ~/.local/share/delegate-runs.jsonl
```

### Why no pseudo-TTY?

Unlike Mistral Vibe, OpenCode CLI works fine in a plain pipe — no TTY check on startup.

### Why prompt via temp file?

Inline shell arguments break when the prompt contains Python dict syntax, emojis, accented characters, or multi-line code. Writing to a temp file avoids all shell injection issues.

### Why a timeout instead of `--max-turns`?

OpenCode CLI has no `--max-turns` flag. The timeout is the only runaway-control lever.

---

## Token economics

OpenCode's internal tool calls consume **OpenCode tokens**, not Claude tokens.
Claude only receives the compressed final output (~200-800 tokens/run).

**Pricing depends on the model used:**
- `opencode/deepseek-v4-flash-free` — **free**, no API key needed (may have queue delays)
- `google/gemini-2.5-flash` — requires `GOOGLE_GENERATIVE_AI_API_KEY` env var

Real token counts and cost are printed after every run and appended to the run log.

---

## Known limitations

| Limitation | Detail |
|-----------|--------|
| No `--max-turns` | Timeout is the only runaway control |
| `--dangerously-skip-permissions` | All tools auto-approved (safe with post-run review) |
| Free model delays | May have queue during peak usage |
| No `--workdir` flag | Script `cd`s into workdir instead |

---

## Run Log

Every run appends one JSON entry to `~/.local/share/delegate-runs.jsonl`.
The shared `~/tools/delegate-report` script can query all runs across delegates.

```bash
~/tools/delegate-report                  # full report
~/tools/delegate-report --since 7        # last 7 days
~/tools/delegate-report --fails          # failures only
```

---

## Sister projects

- [vibe-skill](https://github.com/pcx-wave/vibe-skill) — delegate to Mistral Vibe
- [gemini-skill](https://github.com/pcx-wave/gemini-skill) — delegate to Gemini CLI

All three write to the same `delegate-runs.jsonl` log, making runs comparable across delegates.

---

## License

MIT
