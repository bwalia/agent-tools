# Cursor → claude-mem mirroring (2026-10-12/13)

Cursor prompts and tool activity land in the Mac Studio's claude-mem worker
(127.0.0.1:37701, viewer at the same address) next to Claude Code sessions,
tagged `platform_source=cursor`.

## Mac Studio (Balinders-Mac-Studio.local, 192.168.1.177)

- `claude-mem cursor install user` → `~/.cursor/hooks.json` (local Cursor, if ever installed).
- Worker stays bound to 127.0.0.1 — not exposed on the LAN.
- `~/.ssh/authorized_keys` has one forward-only line for the laptop's tunnel key:
  `restrict,port-forwarding,permitopen="127.0.0.1:37701",command="/usr/bin/false" ssh-ed25519 … claude-mem-tunnel@macbook-pro`
  (backup: `authorized_keys.bak-20261013`).

## MacBook Pro (Balinders-MacBook-Pro-2.local, Cursor.app runs here)

| Path | Purpose |
|---|---|
| `~/.ssh/claude_mem_tunnel` | dedicated ed25519 key, used only for the tunnel |
| `~/Library/LaunchAgents/uk.workstation.claude-mem-tunnel.plist` | `ssh -N -L 127.0.0.1:37701:127.0.0.1:37701` to the Studio, KeepAlive |
| `~/.claude-mem-remote/bin/bun` | copy of the Studio's bun (1.3.14, arm64) |
| `~/.claude-mem-remote/plugin/` | copy of claude-mem 13.36.1 (matches the Studio worker) |
| `~/.claude-mem-remote/bin/hook` | wrapper: `worker-service.cjs hook cursor <phase>` with `CLAUDE_MEM_WORKER_AUTOSTART=false` |
| `~/.cursor/hooks.json` | beforeSubmitPrompt, afterMCPExecution, afterShellExecution, afterFileEdit, stop → wrapper |
| `~/.claude-mem/settings.json` | pinned to 127.0.0.1:37701, AUTOSTART=false (no local worker) |

## Limits

- Tunnel uses the `.local` name, so it only works on the home LAN. Off-LAN, Cursor is
  unaffected but nothing is recorded. Joining the laptop to the WG mesh (10.8.0.18
  reserved for macbook-pro) would fix that.
- The laptop copy does not auto-update. After a claude-mem upgrade on the Studio, re-sync:
  `rsync -a --delete ~/.claude/plugins/cache/thedotmack/claude-mem/<ver>/ Balinders-MacBook-Pro-2.local:.claude-mem-remote/plugin/`

## Checks

    # on the laptop
    curl -s 127.0.0.1:37701/api/health
    tail ~/.claude-mem-remote/tunnel.log
    # on the Studio
    sqlite3 ~/.claude-mem/claude-mem.db "select id, project, started_at from sdk_sessions where platform_source='cursor' order by id desc limit 5;"

## Rollback

    # laptop
    launchctl bootout gui/$(id -u)/uk.workstation.claude-mem-tunnel
    rm ~/Library/LaunchAgents/uk.workstation.claude-mem-tunnel.plist ~/.cursor/hooks.json
    rm -rf ~/.claude-mem-remote ~/.claude-mem ~/.ssh/claude_mem_tunnel*
    # Studio: delete the claude-mem-tunnel@macbook-pro line from ~/.ssh/authorized_keys
