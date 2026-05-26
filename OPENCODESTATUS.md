---
name: opencodestatus
description: Show OpenCode auto-delegate mode status (ON/OFF).
license: MIT
user-invocable: true
allowed-tools:
  - bash
---

# /opencodestatus

Run: `test -f ~/.local/share/opencode-auto.flag && echo "Auto-opencode: ON" || echo "Auto-opencode: OFF"`
