---
name: opencode-model-pick
description: Override the OpenCode model for all subsequent delegations. Usage: /opencode-model-pick <model>
license: MIT
user-invocable: true
allowed-tools:
  - bash
---

# /opencode-model-pick

Run: `echo "<model>" > ~/.local/share/opencode-model.flag`

Then confirm: "OpenCode model set to <model> — all delegations will use this model until cleared."

Available models (from `opencode models`):

| Model | Notes |
|-------|-------|
| `opencode/deepseek-v4-flash-free` | Free, default — no API key needed |
| `opencode/big-pickle` | Free |
| `opencode/nemotron-3-super-free` | Free |
| `google/gemini-2.5-flash` | Requires `GOOGLE_GENERATIVE_AI_API_KEY` |
| `github-copilot/claude-sonnet-4.6` | Requires GitHub Copilot subscription |
