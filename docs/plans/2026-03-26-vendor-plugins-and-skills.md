# Vendor Plugins and Skills for Sandbox Compatibility

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make all Claude Code plugins and custom skills reliably available inside the Docker sandbox by vendoring them into the dev repo and extending `sync.sh` to install them correctly in each environment.

**Architecture:** The dev repo becomes the canonical home for plugin and skill files. Skills are symlinked on the host (edits flow back to git) and copied in the sandbox. Plugins are vendored snapshots — always copied, never symlinked — and updated manually when needed. The `~/.claude:ro` mount is removed from sandbox creation since nothing reads from it.

**Tech Stack:** Bash, jq, Docker sandbox

---

## Context

### The problem

Inside the Docker sandbox, `$HOME` = `/home/agent/`. Claude Code reads config from `$HOME/.claude/`, which is `/home/agent/.claude/`. The host's `~/.claude` is mounted read-only at `/Users/arpanet/.claude/` — a path Claude Code never checks.

The current `sync.sh` copies `settings.json`, `CLAUDE.md`, `statusline-command.sh`, and `commands/` from the dev repo into `/home/agent/.claude/`. But plugins and skills are not synced, so they're invisible to Claude Code in the sandbox.

### Two separate `.claude` directories in the sandbox

| Path | Source | Writable? | Used by Claude Code? |
|------|--------|-----------|---------------------|
| `/Users/arpanet/.claude/` | VirtioFS host mount (`:ro`) | No | No |
| `/home/agent/.claude/` | Container overlay layer | Yes | **Yes** |

### What needs syncing

| Component | Host source | Sandbox target |
|-----------|------------|----------------|
| Plugin cache | `dev/plugins/cache/` | `$HOME/.claude/plugins/cache/` |
| `installed_plugins.json` | Generated from template | `$HOME/.claude/plugins/installed_plugins.json` |
| `config.json` | `dev/plugins/config.json` | `$HOME/.claude/plugins/config.json` |
| Custom skills | `dev/skills/` | `$HOME/.claude/skills/` |

### Host vs sandbox behavior

| Action | Host | Sandbox |
|--------|------|---------|
| Skills | Symlink `~/.claude/skills/<name>` → `dev/skills/<name>` | Copy `dev/skills/<name>` → `$HOME/.claude/skills/<name>` |
| Plugin cache | Copy `dev/plugins/cache/` → `~/.claude/plugins/cache/` | Copy `dev/plugins/cache/` → `$HOME/.claude/plugins/cache/` |
| `installed_plugins.json` | Generated from template with `$HOME` paths | Generated from template with `$HOME` paths |
| `config.json` | Copied | Copied |

Plugins are never symlinked. They're vendored snapshots — updated manually when a new version is needed.

---

## File Structure

### New files in dev repo

```
dev/
  plugins/
    cache/
      superpowers-marketplace/
        superpowers/5.0.1/              # Vendored (full tree, ~50 files)
        elements-of-style/1.0.0/        # Vendored (~5 files)
        double-shot-latte/1.2.0/        # Vendored (~75 files incl. test scenarios)
      claude-plugins-official/
        frontend-design/205b6e0b3036/   # Vendored (~4 files)
    installed_plugins.template.json     # __CLAUDE_HOME__ placeholder in paths
    config.json                         # Static: {"repositories": {}}
  skills/
    deep-clean/          # Already exists
    wrap/                # Already exists
    # obsidian and rtb-draft-analyst are local-only — not vendored
```

### Modified files

```
dev/bin/sync.sh          # Extended Phase 3: skills + plugins
~/.zshrc (line 28)       # Remove ~/.claude:ro from sandbox-create alias
```

### Files NOT vendored (intentionally)

- `~/.claude/plugins/marketplaces/` — git clones used only for discovery/updates, not needed at runtime
- `~/.claude/plugins/blocklist.json` — fetched remotely by Claude Code at startup
- `~/.claude/plugins/install-counts-cache.json` — ephemeral cache
- `~/.claude/plugins/known_marketplaces.json` — only needed for `/plugin update`, not runtime

---

## Chunk 1: Vendor files into dev repo

### Task 1: Vendor plugin cache

**Files:**
- Create: `dev/plugins/cache/` (full tree copied from host)
- Create: `dev/plugins/installed_plugins.template.json`
- Create: `dev/plugins/config.json`

- [ ] **Step 1: Copy plugin cache directories from host into dev repo**

```bash
mkdir -p dev/plugins/cache
cp -r ~/.claude/plugins/cache/superpowers-marketplace dev/plugins/cache/
cp -r ~/.claude/plugins/cache/claude-plugins-official dev/plugins/cache/
```

- [ ] **Step 2: Create `installed_plugins.template.json`**

Template uses `__CLAUDE_HOME__` as a placeholder that `sync.sh` will replace with the correct `$CLAUDE_HOME` path for the environment.

```json
{
  "version": 2,
  "plugins": {
    "superpowers@superpowers-marketplace": [
      {
        "scope": "user",
        "installPath": "__CLAUDE_HOME__/plugins/cache/superpowers-marketplace/superpowers/5.0.1",
        "version": "5.0.1",
        "installedAt": "2026-01-15T15:24:49.321Z",
        "lastUpdated": "2026-03-11T21:56:34.922Z",
        "gitCommitSha": "a08f088968d0789a44a4f5f5f7d8d6bbe46a0b44"
      }
    ],
    "elements-of-style@superpowers-marketplace": [
      {
        "scope": "user",
        "installPath": "__CLAUDE_HOME__/plugins/cache/superpowers-marketplace/elements-of-style/1.0.0",
        "version": "1.0.0",
        "installedAt": "2025-10-28T14:09:59.424Z",
        "lastUpdated": "2025-10-28T14:09:59.424Z",
        "gitCommitSha": "6099c505c2a8eb066f3777f83a97d9d828f7954c"
      }
    ],
    "double-shot-latte@superpowers-marketplace": [
      {
        "scope": "user",
        "installPath": "__CLAUDE_HOME__/plugins/cache/superpowers-marketplace/double-shot-latte/1.2.0",
        "version": "1.2.0",
        "installedAt": "2026-02-11T22:00:46.403Z",
        "lastUpdated": "2026-03-11T21:56:34.663Z",
        "gitCommitSha": "914cff0afc72c534fe91fb258f0875f212368619"
      }
    ],
    "frontend-design@claude-plugins-official": [
      {
        "scope": "user",
        "installPath": "__CLAUDE_HOME__/plugins/cache/claude-plugins-official/frontend-design/205b6e0b3036",
        "version": "205b6e0b3036",
        "installedAt": "2026-03-19T19:29:03.740Z",
        "lastUpdated": "2026-03-19T19:29:03.740Z",
        "gitCommitSha": "205b6e0b30366a969412d9aab7b99bea99d58db1"
      }
    ]
  }
}
```

- [ ] **Step 3: Create `config.json`**

```json
{
  "repositories": {}
}
```

- [ ] **Step 4: Verify plugin trees are complete**

Spot-check that vendored trees have the expected structure (SKILL.md files, plugin.json, hooks, etc.).

Note: `obsidian` and `rtb-draft-analyst` are local-only skills — not vendored, not needed in the sandbox. They stay where they are (iCloud vault and `rtb-mcp-server/skill/` respectively).

---

## Chunk 2: Extend sync.sh

### Task 3: Add directory sync support to sync.sh

**Files:**
- Modify: `dev/bin/sync.sh`

- [ ] **Step 1: Add `link_or_copy_dir` function**

Similar to existing `link_or_copy` but handles directories (uses `cp -r` in sandbox mode, `ln -s` on host). Place it right after the existing `link_or_copy` function.

```bash
link_or_copy_dir() {
  local source="${1%/}"
  local target="${2%/}"
  local label="$3"

  mkdir -p "$(dirname "$target")"

  if [ "$IS_SANDBOX" = true ]; then
    rm -rf "$target"
    cp -r "$source" "$target"
    echo -e "  ${GREEN}✓${NC} $label copied"
    return
  fi

  if [ -L "$target" ]; then
    current=$(readlink "$target")
    if [ "$current" = "$source" ]; then
      echo -e "  ${GREEN}✓${NC} $label already linked"
      return
    else
      echo -e "  ${YELLOW}⚠${NC} $label points to $current, updating..."
      rm "$target"
    fi
  elif [ -d "$target" ]; then
    echo -e "  ${YELLOW}⚠${NC} $label exists as directory, replacing with link..."
    rm -rf "$target"
  fi

  ln -s "$source" "$target"
  symlinks_created_count=$((symlinks_created_count + 1))
  echo -e "  ${GREEN}✓${NC} $label → $source"
}
```

- [ ] **Step 2: Add skills sync block**

After the commands sync block, add:

```bash
# Sync skills
if [ -d "$DEV_REPO/skills" ]; then
  mkdir -p "$CLAUDE_HOME/skills"
  for skill_dir in "$DEV_REPO/skills"/*/; do
    [ -d "$skill_dir" ] || continue
    skill_name="$(basename "$skill_dir")"
    link_or_copy_dir "$skill_dir" "$CLAUDE_HOME/skills/$skill_name" "skills/$skill_name"
  done
fi
```

- [ ] **Step 3: Add plugin sync block (copy in all environments)**

Plugins are vendored snapshots — always copied, never symlinked.

```bash
# Sync plugins (always copy — vendored snapshots, not live-edited)
if [ -d "$DEV_REPO/plugins/cache" ]; then
  for marketplace_dir in "$DEV_REPO/plugins/cache"/*/; do
    [ -d "$marketplace_dir" ] || continue
    marketplace="$(basename "$marketplace_dir")"
    for plugin_dir in "$marketplace_dir"/*/; do
      [ -d "$plugin_dir" ] || continue
      plugin="$(basename "$plugin_dir")"
      for version_dir in "$plugin_dir"/*/; do
        [ -d "$version_dir" ] || continue
        version="$(basename "$version_dir")"
        target="$CLAUDE_HOME/plugins/cache/$marketplace/$plugin/$version"
        mkdir -p "$(dirname "$target")"
        rm -rf "$target"
        cp -r "${version_dir%/}" "$target"
        echo -e "  ${GREEN}✓${NC} plugins/$plugin/$version copied"
      done
    done
  done
fi
```

- [ ] **Step 4: Add `installed_plugins.json` and config generation (both environments)**

```bash
# Generate installed_plugins.json from template
if [ -f "$DEV_REPO/plugins/installed_plugins.template.json" ]; then
  mkdir -p "$CLAUDE_HOME/plugins"
  sed "s|__CLAUDE_HOME__|$CLAUDE_HOME|g" "$DEV_REPO/plugins/installed_plugins.template.json" \
    > "$CLAUDE_HOME/plugins/installed_plugins.json"
  echo -e "  ${GREEN}✓${NC} installed_plugins.json generated"
fi

# Copy static plugin config
if [ -f "$DEV_REPO/plugins/config.json" ]; then
  mkdir -p "$CLAUDE_HOME/plugins"
  cp "$DEV_REPO/plugins/config.json" "$CLAUDE_HOME/plugins/config.json"
  echo -e "  ${GREEN}✓${NC} plugins/config.json copied"
fi
```

- [ ] **Step 5: Update summary output**

Add skills and plugins to the summary line:

```bash
echo -e "  Symlinks verified: settings.json, CLAUDE.md, statusline-command.sh, commands/, skills/, plugins/"
```

---

## Chunk 3: Update host config and documentation

### Task 4: Update .zshrc sandbox alias

**Files:**
- Modify: `~/.zshrc` (line 28)

- [ ] **Step 1: Remove `~/.claude:ro` from sandbox-create alias**

Change:
```bash
alias sandbox-create="docker sandbox create -t dev-template:latest claude ~/dev ~/.claude:ro"
```
To:
```bash
alias sandbox-create="docker sandbox create -t dev-template:latest claude ~/dev"
```

### Task 5: Update Obsidian doc

**Files:**
- Modify: Obsidian vault `Misc/Docker Sandbox for Claude Code.md`

- [ ] **Step 1: Update sandbox creation command** (step 3 and aliases)

Remove `~/.claude:ro` from all sandbox creation examples.

- [ ] **Step 2: Replace step 7 (plugin install)** with note about sync.sh

Plugins are now vendored in the dev repo and synced by `sync.sh`. No separate install step needed.

- [ ] **Step 3: Update the "Workspace model" section**

Note that `~/.claude` is no longer mounted. All Claude config comes from the dev repo via `sync.sh`.

### Task 6: Run sync.sh on the host and verify

- [ ] **Step 1: Run sync.sh**

```bash
~/code/dev/bin/sync.sh
```

This will:
- Replace existing skill symlinks with ones pointing to the dev repo
- Copy vendored plugin cache into `~/.claude/plugins/cache/`
- Generate `installed_plugins.json` with correct host paths

- [ ] **Step 2: Verify Claude Code still works on host**

Launch `claude` and confirm all skills and plugins load correctly.

- [ ] **Step 3: Commit vendored files to dev repo**

```bash
cd ~/code/dev
git add plugins/ skills/obsidian/ docs/plans/
git commit -m "Vendor plugins and skills for sandbox compatibility"
```
