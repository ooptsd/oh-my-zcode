---
description: ""
---

# omz wiki

This compatibility command keeps `/omz:wiki` available without loading the full `wiki` skill description in every Claude Code session.

## Dispatch

1. Read the full bundled skill instructions from the active omz plugin/install: `skills/wiki/SKILL.md`.
2. Follow that SKILL.md exactly, treating the user's arguments as:

```text
$ARGUMENTS
```

If the file is not directly readable from the current working directory, locate it under the active `CLAUDE_PLUGIN_ROOT`/`OMC_PLUGIN_ROOT`, package root, or installed omz plugin directory, then continue.
