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
git clone https://github.com/pcx-wave/opencode-skill.git && cd opencode-skill && mkdir -p ~/tools ~/.claude/skills/opencode ~/.claude/skills/opencodeon ~/.claude/skills/opencodeoff ~/.claude/skills/opencodestatus ~/.claude/skills/opencode-model-pick ~/.claude/skills/opencode-model-clear && ln -sf "$(pwd)/tools/opencode-delegate" ~/tools/opencode-delegate && ln -sf "$(pwd)/tools/delegate-report" ~/tools/delegate-report && chmod +x ~/tools/opencode-delegate ~/tools/delegate-report && ln -sf "$(pwd)/SKILL.md" ~/.claude/skills/opencode/SKILL.md && ln -sf "$(pwd)/OPENCODEON.md" ~/.claude/skills/opencodeon/SKILL.md && ln -sf "$(pwd)/OPENCODEOFF.md" ~/.claude/skills/opencodeoff/SKILL.md && ln -sf "$(pwd)/OPENCODESTATUS.md" ~/.claude/skills/opencodestatus/SKILL.md && ln -sf "$(pwd)/OPENCODEMODELPICK.md" ~/.claude/skills/opencode-model-pick/SKILL.md && ln -sf "$(pwd)/OPENCODEMODELCLEAR.md" ~/.claude/skills/opencode-model-clear/SKILL.md
```

### Step-by-step

```bash
# 1. Clone this repo
git clone https://github.com/pcx-wave/opencode-skill.git
cd opencode-skill

# 2. Install the scripts (symlinks — stay in sync with git pull)
mkdir -p ~/tools
ln -sf "$(pwd)/tools/opencode-delegate" ~/tools/opencode-delegate
ln -sf "$(pwd)/tools/delegate-report" ~/tools/delegate-report
chmod +x ~/tools/opencode-delegate ~/tools/delegate-report

# 3. Install the skills for Claude Code
mkdir -p ~/.claude/skills/opencode ~/.claude/skills/opencodeon ~/.claude/skills/opencodeoff \
         ~/.claude/skills/opencodestatus ~/.claude/skills/opencode-model-pick ~/.claude/skills/opencode-model-clear
ln -sf "$(pwd)/SKILL.md" ~/.claude/skills/opencode/SKILL.md
ln -sf "$(pwd)/OPENCODEON.md" ~/.claude/skills/opencodeon/SKILL.md
ln -sf "$(pwd)/OPENCODEOFF.md" ~/.claude/skills/opencodeoff/SKILL.md
ln -sf "$(pwd)/OPENCODESTATUS.md" ~/.claude/skills/opencodestatus/SKILL.md
ln -sf "$(pwd)/OPENCODEMODELPICK.md" ~/.claude/skills/opencode-model-pick/SKILL.md
ln -sf "$(pwd)/OPENCODEMODELCLEAR.md" ~/.claude/skills/opencode-model-clear/SKILL.md

# 4. (Optional) Enable auto-mode — Claude delegates all code tasks automatically
#    without requiring /opencode each time. Toggle with /opencodeon and /opencodeoff.
grep -q "opencode auto-mode" ~/.claude/CLAUDE.md 2>/dev/null || cat >> ~/.claude/CLAUDE.md << 'EOF'

# opencode auto-mode
At the start of every user request that involves writing, editing, or fixing code:
1. Run `test -f ~/.local/share/opencode-auto.flag` (silent, no output to user).
2. If the flag exists → automatically invoke the `opencode` skill exactly as if the user had typed `/opencode <their full instruction>`. Do NOT ask first, do NOT explain — just delegate.
3. If the flag is absent → proceed normally.

The flag is toggled by `/opencodeon` and `/opencodeoff`.
EOF
```

### Verify the install

```bash
# Check opencode is available
opencode --version

# Test the delegate script
~/tools/opencode-delegate /tmp "Say hello in one sentence." 60
# Should print: [opencode] Hello! ...
```

### Updating

Because the install uses symlinks, a `git pull` is all you need:

```bash
cd ~/opencode-skill && git pull
```

`~/tools/opencode-delegate` and all skill files are automatically up to date — no re-copy needed.

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
