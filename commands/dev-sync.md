# /dev-sync

Synchronize shared Claude Code configuration across all projects.

## What this does

1. **Gathers** permissions from all project `.claude/settings.local.json` files in `~/code/`
2. **Promotes** generic tool permissions to the global `settings.json` (project-specific ones stay local)
3. **Trims** promoted entries from project local files to keep them clean
4. **Verifies** symlinks from `~/.claude/` point to the dev repo

## Promotion rules

- Generic CLI tools (git, npm, pnpm, node, turso, etc.) → always promote
- Skills, MCP tools, WebSearch → always promote
- WebFetch domains → promote if found in 2+ projects
- Project-specific commands (absolute paths, env prefixes, `./` scripts) → keep local
- `additionalDirectories` (except `/tmp`) → keep local

## Run it

```bash
$HOME/code/dev/bin/sync.sh
```

Review the summary output and verify changes look correct. If this is the first run, existing files at `~/.claude/settings.json` and `~/.claude/CLAUDE.md` will be backed up to `.bak` before symlinking.
