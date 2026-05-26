---
name: opencodeoff
description: Disable OpenCode auto-delegate mode — coding tasks are handled by Claude directly unless /opencode is explicitly invoked.
license: MIT
user-invocable: true
allowed-tools:
  - bash
---

# /opencodeoff

Run: `rm -f ~/.local/share/opencode-auto.flag`

Then confirm: "Auto-opencode OFF — Claude will handle coding tasks directly."
