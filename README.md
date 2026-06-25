# Claude Code Sync

Sync your [Claude Code](https://claude.com/claude-code) CLI config across machines with [Syncthing](https://syncthing.net/). macOS, Linux, Windows.

Claude Code has no built-in sync (as of March 2026). This repo has a tested `.stignore` whitelist that syncs only the portable config files and skips runtime data.

## What Syncs

| Path | What |
|------|------|
| `CLAUDE.md` | Global instructions, loaded every conversation |
| `skills/` | Custom slash commands (`/ship`, `/sync`, etc.) |
| `agents/` | [Subagent](https://code.claude.com/docs/en/sub-agents) definitions |
| `agent-memory/` | Persistent subagent memory (user scope) |
| `rules/` | User-level [rules](https://code.claude.com/docs/en/memory) applied before project rules |
| `keybindings.json` | Custom [keybindings](https://code.claude.com/docs/en/keybindings) |
| `statusline.js` | Cross-platform Node.js statusline script (see note below) |
| `git-hooks/` | gitleaks pre-commit secret scanner (needs per-machine setup, see note below) |
| `memory/` | Global memory files |
| `commands/` | Legacy commands (use `skills/` instead) |
| `projects/*/memory/` | Per-project memory (MEMORY.md + topic files) |
| `projects/*/CLAUDE.md` | Per-project instructions |
| `CLAUDE.md.bak-*` | CLAUDE.md backups |

## What Does NOT Sync

| Path | Why |
|------|-----|
| `settings.json` | Per-machine plugins, env vars, MCP servers |
| `settings.local.json` | Per-machine permissions |
| `.credentials.json` | Auth tokens |
| `history.jsonl` | Session transcripts (large) |
| `plans/`, `tasks/`, `todos/` | Session-bound state |
| `plugins/`, `teams/` | Per-machine runtime |
| `statusline.sh` | OS-specific shell commands (use cross-platform `statusline.js` instead) |
| `cache/`, `debug/`, `downloads/` | Runtime data |
| `file-history/`, `backups/` | Auto-generated snapshots |
| `session-env/`, `shell-snapshots/` | Ephemeral state |
| `usage-data/`, `telemetry/` | Analytics (can be huge) |
| `ide/`, `chrome/`, `paste-cache/` | Tool runtime |
| `projects/**/*.jsonl`, `*.meta.json` | Conversation logs |
| `projects/**/subagents/`, `tool-results/` | Subagent runtime |

### Statusline note

`statusline.js` syncs, but `settings.json` does not — so on each new machine you must also add this block to `~/.claude/settings.json` once:

```json
"statusLine": { "type": "command", "command": "node ~/.claude/statusline.js" }
```

The script is portable across macOS, Linux, and Windows. Requires Node.js on PATH.

### Git secret-scanning hook

`git-hooks/pre-commit` syncs, but the git config that activates it and the executable bit do **not**. On each machine, run the one-time setup:

```bash
# 1. install gitleaks
brew install gitleaks            # macOS
# sudo apt install gitleaks      # Debian/Ubuntu
# winget install gitleaks.gitleaks   # Windows

# 2. point all repos at the synced hooks dir
git config --global core.hooksPath ~/.claude/git-hooks

# 3. make the hook executable (re-run if Syncthing later rewrites the file)
chmod +x ~/.claude/git-hooks/pre-commit
```

The hook scans staged changes with gitleaks and blocks the commit on a hit. It fails open if gitleaks is not installed (commits proceed unscanned), so step 1 is what actually provides protection. A repo that needs its own pre-commit hook can place it at `.git/hooks/pre-commit.local` — the global hook will run it after the secret scan. Bypass a single commit with `git commit --no-verify`.

This is a local backstop. GitHub push protection (enabled per-repo in Settings -> Code security) is the server-side net that also catches anything committed with `--no-verify`.

## Setup

### Prerequisites

- [Syncthing](https://syncthing.net/) on all machines
- [Claude Code CLI](https://claude.com/claude-code) on all machines

### 1. Copy `.stignore` to every machine

Put this at `~/.claude/.stignore`. Syncthing does **not** sync its own `.stignore`, so you need to copy it manually to each machine.

A copy is in this repo: [`.stignore`](.stignore)

```
// Syncthing ignore file for ~/.claude/
// Whitelist approach: first match wins.
// Both !/dir and !/dir/** are needed: the first un-ignores
// the directory itself, the second un-ignores its contents.
//
// SAFE to sync:
!/CLAUDE.md
!/CLAUDE.md.bak-*
!/skills
!/skills/**
!/agents
!/agents/**
!/agent-memory
!/agent-memory/**
!/keybindings.json
!/statusline.js
!/memory
!/memory/**
!/commands
!/commands/**
!/rules
!/rules/**
!/.stignore
!/git-hooks
!/git-hooks/**
//
// Block runtime data inside projects (must come before !/projects)
projects/**/*.jsonl
projects/**/*.meta.json
projects/**/subagents
projects/**/subagents/**
projects/**/tool-results
projects/**/tool-results/**
//
// Allow project memory and CLAUDE.md only
!/projects
!/projects/*
!/projects/*/memory
!/projects/*/memory/**
!/projects/*/CLAUDE.md
//
// Block everything else
*
```

The `*` at the bottom blocks everything not explicitly allowed. First match wins, so the project block rules must come before `!/projects`. New directories added by future Claude Code updates are blocked by default.

### 2. Add the folder in Syncthing

Open `http://127.0.0.1:8384` and add a shared folder:

| Setting | Value |
|---------|-------|
| Folder Label | `Claude Code Sync` |
| Folder ID | `claude-code-sync` |
| Folder Path | `~/.claude` (macOS/Linux) or `C:\Users\<you>\.claude` (Windows) |
| File Versioning | Staggered, 30 days |
| **Ignore Delete** | **`true`** |
| Ignore Permissions | `true` |
| File Pull Order | `smallestFirst` |
| Watch for Changes | `true` |

On each additional machine:

1. Add the remote device
2. Accept the `claude-code-sync` folder, set path to `~/.claude`
3. Set **Ignore Delete** to `true`
4. Set **Staggered Versioning**, 30 days
5. Copy `.stignore` to `~/.claude/.stignore`

### 3. Verify

```bash
# macOS
API_KEY=$(grep apikey ~/Library/Application\ Support/Syncthing/config.xml | sed 's/.*>\(.*\)<.*/\1/')
curl -s -X POST -H "X-API-Key: $API_KEY" "http://127.0.0.1:8384/rest/db/scan?folder=claude-code-sync"

# Linux
API_KEY=$(grep apikey ~/.local/state/syncthing/config.xml | sed 's/.*>\(.*\)<.*/\1/')
curl -s -X POST -H "X-API-Key: $API_KEY" "http://127.0.0.1:8384/rest/db/scan?folder=claude-code-sync"
```

Check the other machine:

```bash
ls -la ~/.claude/CLAUDE.md
ls ~/.claude/skills/*/SKILL.md 2>/dev/null
```

```cmd
:: Windows
if exist C:\Users\%USERNAME%\.claude\CLAUDE.md echo SYNCED
```

## Key Settings

### Ignore Delete

Set this to `true` on **every device**. If one machine's config gets wiped (Claude Code update, accidental delete), the others keep their copies. Without this, a delete on one machine cascades everywhere.

### Staggered Versioning

Keeps timestamped backups of changed/deleted files in `.stversions/` for 30 days.

```bash
ls ~/.claude/.stversions/
cp ~/.claude/.stversions/CLAUDE~20260307-080132.md ~/.claude/CLAUDE.md
```

### Why settings.json is excluded

Each machine needs its own `settings.json`: plugins, env vars, status line commands, and MCP server configs all differ per OS and machine. Configure it separately everywhere.

## CLAUDE.md Load Order

1. Managed policy (enterprise)
2. **`~/.claude/CLAUDE.md`** (user-level, this is what we sync)
3. `CLAUDE.md` in project root (shared via git)
4. `CLAUDE.local.md` (git-ignored, per-machine overrides)

## Project Memory: Cross-OS Caveat

Project memory paths are derived from absolute filesystem paths. The same repo gets different identifiers on different OSes (`-Users-andrew-repos-myapp` on macOS vs `F--projects-myapp` on Windows). Claude Code only reads the directory matching the local path.

Cross-OS project memory won't carry over automatically. Still worth syncing for:
- Same-OS machines with matching paths (works directly)
- Disaster recovery (rebuild a machine, memory is there)
- Manual reference across machines

## Directory Layout

```
~/.claude/
├── CLAUDE.md                    SYNCED
├── .stignore                    MANUAL (copy to each machine)
├── keybindings.json             SYNCED
├── statusline.js                SYNCED (needs settings.json registration)
├── git-hooks/                   SYNCED (needs core.hooksPath + chmod +x per machine)
│   └── pre-commit
├── skills/                      SYNCED
│   ├── ship/SKILL.md
│   └── .../
├── agents/                      SYNCED
├── agent-memory/                SYNCED
├── rules/                       SYNCED
├── memory/                      SYNCED
├── commands/                    SYNCED (legacy)
├── projects/                    PARTIAL
│   └── <project>/
│       ├── memory/              SYNCED
│       ├── CLAUDE.md            SYNCED
│       ├── *.jsonl              blocked
│       └── subagents/           blocked
├── settings.json                blocked (per-machine)
├── settings.local.json          blocked (per-machine)
├── .credentials.json            blocked (secrets)
├── plans/                       blocked
├── tasks/                       blocked
├── plugins/                     blocked
├── cache/, debug/, ...          blocked
├── .stfolder/                   Syncthing marker
└── .stversions/                 Syncthing backups
```

## Troubleshooting

**Files not syncing:** Check connections in the Syncthing UI. Verify `.stignore` is identical on all machines (it's not auto-synced). Trigger a rescan with the curl commands above.

**Files deleted everywhere:** Check `.stversions/` for timestamped backups. Copy the file back and Syncthing will propagate it.

**Sync conflict files:** Files like `settings.sync-conflict-*.json` appear when two machines edit the same file. Safe to delete.

**Windows: can't find `.claude`:** It's hidden. Use `dir /a` or `if exist C:\Users\<you>\.claude\NUL echo EXISTS`.

## Cross-OS Paths

| OS | `~/.claude` | Syncthing Config |
|----|-------------|------------------|
| macOS | `/Users/<you>/.claude` | `~/Library/Application Support/Syncthing/config.xml` |
| Linux | `/home/<you>/.claude` | `~/.local/state/syncthing/config.xml` |
| Windows | `C:\Users\<you>\.claude` | `%LOCALAPPDATA%\Syncthing\config.xml` |

## Alternatives

| Approach | Tradeoff |
|----------|----------|
| **Syncthing** (this repo) | Real-time, selective, P2P, encrypted. Needs Syncthing everywhere. |
| **Dropbox/iCloud + symlinks** | Simple but no selective sync, risk of corrupting runtime files. |
| **[CCMS](https://github.com/miwidot/ccms)** | rsync over SSH. Manual, not real-time. |
| **[claude-code-config-sync](https://www.npmjs.com/package/claude-code-config-sync)** | npm package. Extra dependency. |
| **Git repo** | Version controlled. Manual commit/push/pull. |

## Related

- **[Claude Insights Merge](https://github.com/andrewle8/claude-insights-merge)** — Merge `/insights` data across multiple machines into a single combined report. Complements this tool: Sync handles config, Insights Merge handles analytics.

## Security

- All transfers are TLS encrypted
- P2P by default. Relay servers are used as fallback but data is end-to-end encrypted (relay can't read it)
- `.credentials.json` and `settings.local.json` are blocked by the catch-all rule
- `settings.json` is excluded too (may contain MCP API keys)
- Check your `CLAUDE.md` for internal paths before making it public

## License

MIT
