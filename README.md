# Claude Code Sync

Sync your Claude Code CLI configuration across multiple machines using [Syncthing](https://syncthing.net/). Works across macOS, Linux, and Windows.

## The Problem

Claude Code stores all configuration in `~/.claude/` — your global `CLAUDE.md`, custom skills, agent definitions, project memory, keybindings, and more. When you use Claude Code on multiple machines, each one starts from scratch with no shared context.

Manually copying configs is tedious and error-prone. Cloud sync tools like Dropbox or Google Drive can corrupt files with lock conflicts. What you need is **selective, bidirectional, real-time sync** that only touches the safe config files and leaves runtime data alone.

## The Solution

Syncthing provides encrypted, peer-to-peer sync with no cloud middleman. Combined with a carefully crafted `.stignore` whitelist, it syncs only the files that are safe to share while ignoring machine-specific runtime data.

### What Syncs

| Path | Description |
|------|-------------|
| `CLAUDE.md` | Global instructions loaded into every conversation |
| `skills/` | Custom slash commands (`/ship`, `/sync`, `/bump-version`, etc.) |
| `agents/` | Custom agent definitions |
| `agent-memory/` | Accumulated agent knowledge across conversations |
| `keybindings.json` | Custom keyboard shortcuts |
| `memory/` | Global memory files |
| `commands/` | Custom commands |
| `projects/*/memory/` | Per-project memory (MEMORY.md and topic files) |
| `projects/*/CLAUDE.md` | Per-project instructions |
| `.stignore` | The ignore file itself (so all machines stay in sync) |

### What Does NOT Sync (Intentionally)

| Path | Why |
|------|-----|
| `settings.json` | Each machine may have different plugins, env vars, or statusline configs |
| `settings.local.json` | Machine-specific permissions |
| `credentials.json` | Auth tokens — never sync secrets |
| `history.jsonl` | Session transcripts (large, machine-specific) |
| `cache/`, `debug/`, `downloads/` | Runtime data |
| `session-env/`, `shell-snapshots/` | Ephemeral session state |
| `projects/**/*.jsonl` | Conversation logs |
| `projects/**/subagents/` | Subagent runtime data |
| `projects/**/tool-results/` | Cached tool outputs |

## Setup

### Prerequisites

- [Syncthing](https://syncthing.net/) installed on all machines
- Claude Code CLI installed on all machines (`npm install -g @anthropic-ai/claude-code`)

### 1. Create the `.stignore` File

Place this file at `~/.claude/.stignore` on your **first** machine. Once Syncthing is configured, it will propagate to other machines automatically.

```
// Syncthing ignore file for ~/.claude/
// Whitelist approach: first match wins in Syncthing.
// Note: !/dir auto-expands to !/dir + !/dir/** so we must
// put specific ignores BEFORE broad un-ignores.
//
// SAFE to sync (config files, no runtime writes):
!/CLAUDE.md
!/CLAUDE.md.bak-*
!/skills
!/skills/**
!/agents
!/agents/**
!/agent-memory
!/agent-memory/**
!/keybindings.json
!/memory
!/memory/**
!/commands
!/commands/**
// settings.json intentionally NOT synced — each machine has different plugins/env
!/.stignore
//
// Block session transcripts and runtime data inside projects
// (these MUST come before !/projects to win first-match)
projects/**/*.jsonl
projects/**/*.meta.json
projects/**/subagents
projects/**/subagents/**
projects/**/tool-results
projects/**/tool-results/**
//
// Allow projects directory structure, memory, and CLAUDE.md
!/projects
!/projects/*
!/projects/*/memory
!/projects/*/memory/**
!/projects/*/CLAUDE.md
//
// IGNORE everything else at root level
*
```

> **How it works:** The `*` at the bottom ignores everything by default. Lines starting with `!` un-ignore specific paths. Syncthing uses first-match-wins, so the explicit block rules for `projects/**/*.jsonl` etc. must come before `!/projects`.

### 2. Configure Syncthing

#### Add the shared folder

On your first machine, open Syncthing's web UI (`http://127.0.0.1:8384`) and add a new folder:

| Setting | Value |
|---------|-------|
| **Folder Label** | `Claude Code Sync` |
| **Folder ID** | `claude-code-sync` |
| **Folder Path** | `~/.claude` (macOS/Linux) or `C:\Users\<you>\.claude` (Windows) |
| **File Versioning** | Staggered, 30 days (see below) |
| **Ignore Delete** | `true` (critical — see below) |
| **Ignore Permissions** | `true` (recommended for cross-OS sync) |
| **File Pull Order** | `smallestFirst` (configs are small, sync them fast) |
| **Watch for Changes** | `true` |

#### On each additional machine

1. Add the remote device in Syncthing
2. Accept or manually add the `claude-code-sync` folder
3. Set the folder path to `~/.claude` (or the Windows equivalent)
4. **Set Ignore Delete to `true`** on every device
5. **Set File Versioning to Staggered** with at least 30 days retention

### 3. Verify

After setup, trigger a rescan and check that files propagate:

```bash
# On any machine with curl
API_KEY=$(grep apikey ~/Library/Application\ Support/Syncthing/config.xml | sed 's/.*>\(.*\)<.*/\1/')
curl -s -X POST -H "X-API-Key: $API_KEY" "http://127.0.0.1:8384/rest/db/scan?folder=claude-code-sync"
```

Then verify on the other machine:

```bash
ls -la ~/.claude/CLAUDE.md
ls ~/.claude/skills/*/SKILL.md
```

## Recommended Settings

### Ignore Delete (Critical)

Set `ignoreDelete: true` on **every device** for the `claude-code-sync` folder. This prevents accidental deletions from propagating across all your machines. If one machine's config gets wiped (e.g., by a Claude Code update), the other machines keep their copies.

Without this, a deletion on any device will cascade to all others.

### Staggered File Versioning

Enable **Staggered File Versioning** with at least 30 days of retention. This keeps timestamped backups of overwritten or deleted files in `.stversions/`. If something goes wrong, you can recover:

```bash
# See what's been backed up
ls ~/.claude/.stversions/

# Recover a deleted CLAUDE.md
cp ~/.claude/.stversions/CLAUDE~20260307-080132.md ~/.claude/CLAUDE.md
```

### Settings.json: Keep It Per-Machine

The `.stignore` intentionally excludes `settings.json` because it contains machine-specific configuration:

- **Plugins**: Not all plugins work on all OSes
- **Environment variables**: Paths differ between machines
- **Status line**: Shell commands are OS-specific
- **MCP servers**: May reference local paths or ports

Configure `settings.json` separately on each machine.

## Directory Layout

```
~/.claude/
├── CLAUDE.md                    ← SYNCED: global instructions
├── .stignore                    ← SYNCED: this ignore file
├── keybindings.json             ← SYNCED: keyboard shortcuts
├── settings.json                ← NOT SYNCED: per-machine config
├── settings.local.json          ← NOT SYNCED: per-machine permissions
├── .credentials.json            ← NOT SYNCED: auth tokens
├── history.jsonl                ← NOT SYNCED: session history
├── skills/                      ← SYNCED
│   ├── ship/SKILL.md
│   ├── sync/SKILL.md
│   └── .../
├── agents/                      ← SYNCED: agent definitions
├── agent-memory/                ← SYNCED: agent accumulated knowledge
├── memory/                      ← SYNCED: global memory
├── commands/                    ← SYNCED: custom commands
├── projects/                    ← PARTIALLY SYNCED
│   ├── <project>/
│   │   ├── memory/MEMORY.md     ← SYNCED
│   │   ├── CLAUDE.md            ← SYNCED
│   │   ├── *.jsonl              ← NOT SYNCED: transcripts
│   │   └── subagents/           ← NOT SYNCED: runtime
│   └── .../
├── .stfolder/                   ← Syncthing marker
├── .stversions/                 ← Syncthing backup copies
├── cache/                       ← NOT SYNCED
├── debug/                       ← NOT SYNCED
└── ...
```

## Troubleshooting

### Files not syncing

1. **Check Syncthing connections**: All devices should show "Connected" in the web UI
2. **Trigger a rescan**: `curl -X POST -H "X-API-Key: $KEY" "http://127.0.0.1:8384/rest/db/scan?folder=claude-code-sync"`
3. **Check the `.stignore`**: Make sure the file you expect to sync is whitelisted
4. **Check completion**: Look at the folder status in the Syncthing UI — it should show 100% for synced items

### Files were deleted across all machines

This is why `ignoreDelete` and staggered versioning exist:

1. Check `.stversions/` on any machine for timestamped backups
2. Copy the file back to the active location
3. Syncthing will propagate the restore

### Sync conflict files

Files like `settings.sync-conflict-20260307-130843-DEVICE.json` appear when two machines modify the same file simultaneously. These are safe to delete — Syncthing keeps both versions so you can merge manually if needed.

### Windows: `.claude` directory appears missing

The `.claude` directory is hidden on Windows. Use `dir /a` to see it, or check with `if exist C:\Users\<you>\.claude\NUL echo EXISTS`.

## Cross-OS Considerations

| OS | `~/.claude` Path | Syncthing Config |
|----|-------------------|------------------|
| macOS | `/Users/<you>/.claude` | `~/Library/Application Support/Syncthing/config.xml` |
| Linux | `/home/<you>/.claude` | `~/.local/state/syncthing/config.xml` or `~/.config/syncthing/config.xml` |
| Windows | `C:\Users\<you>\.claude` | `%LOCALAPPDATA%\Syncthing\config.xml` |

Project memory paths like `projects/-Users-andrewle/memory/` are derived from the working directory path. These will sync across machines even though the paths differ — Claude Code uses these as project identifiers, and having the memories available on any machine means context carries over.

## Security Notes

- Syncthing uses TLS encryption for all transfers — no data is sent in plaintext
- No cloud relay stores your data (direct peer-to-peer, or encrypted relay if direct connection fails)
- `.credentials.json` is explicitly excluded from sync
- `settings.local.json` (which contains tool permission allow-lists) is excluded
- Review your `CLAUDE.md` before making the repo public — it may contain internal paths or server addresses

## License

MIT
