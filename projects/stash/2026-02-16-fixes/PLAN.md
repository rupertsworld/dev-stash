# Stash Fixes Plan

Fixes from `claude/write-system-spec-wRmrY` branch, assessed against current main.

## Implemented Fixes

All high and medium priority fixes implemented. 215 tests passing.

| Fix | Description | Status |
|-----|-------------|--------|
| #13 | Reconciler scan on startup | ✅ Done |
| #14 | MCP list/glob filesystem consistency | ✅ Done |
| #10 | Stash name validation | ✅ Done |
| #7  | Daemon MCP race condition | ✅ Done |
| #2  | Throttled reload | ✅ Done |
| #1  | Dirty flag generation counter | ✅ Done |
| #11 | Config file permissions | ✅ Done |
| #12 | Remove dead inquirer dependency | ✅ Done |
| #6  | setContent uses splice | ✅ Done |
| #9  | PID file for daemon stop | ✅ Done |
| #4  | Parallel GitHub fetch | ⏭️ Skipped (low priority optimization) |

## Already Fixed / Not Needed

- **Fix #3**: Merge logic in provider → Already in main (`mergeWithRemote()`)
- **Fix #8**: promptSecret masking → Already uses `@inquirer/prompts` `password()`

## Summary of Changes

### Security
- **Name validation**: Stash names must be alphanumeric with dots/hyphens/underscores, 1-64 chars
- **Config permissions**: `config.json` written with 0o600, directory with 0o700

### Correctness
- **Reconciler scan**: Daemon now calls `scan()` after `start()` to import existing files
- **MCP consistency**: `stash_list` and `stash_glob` now read from filesystem like `stash_read`
- **Daemon race**: Fresh MCP server created per request to avoid transport collision
- **Dirty flag**: Generation counter ensures flag clears correctly for local-only stashes

### Performance
- **Throttled reload**: `reloadIfStale()` limits reloads to once per 2 seconds
- **Splice API**: `setContent` and `applyPatch` use efficient `splice()` instead of spreading chars

### Portability
- **PID file**: Daemon writes `daemon.pid`, stop command reads it (no `lsof` dependency)

### Cleanup
- **Removed inquirer**: Dead dependency removed from package.json
