# Stash Fixes Plan

Fixes from `claude/write-system-spec-wRmrY` branch, assessed against current main.

## Fixes to Implement

### High Priority

#### ✅ Fix #13: Reconciler Doesn't Scan on Startup (Correctness) - DONE
**File**: `src/daemon.ts`

Daemon starts reconciler but never calls `scan()`. Existing files on disk are invisible to Automerge structure until modified.

```typescript
// Fixed - added scan() call after start()
const reconciler = new StashReconciler(stash);
await reconciler.start();
await reconciler.scan();  // Import existing files
```

**Tests**: `test/daemon.test.ts` - 2 tests added

---

#### Fix #14: MCP list/glob Inconsistent with read/write (Correctness)
**Files**: `src/mcp.ts`

MCP tools are inconsistent:
- `stash_read` → reads from **filesystem**
- `stash_write` → writes to **filesystem**
- `stash_list` → reads from **Automerge structure doc**
- `stash_glob` → reads from **Automerge structure doc**

Without reconciler running (or if scan hasn't happened), list/glob don't see filesystem state.

**Fix**: Make list/glob also read from filesystem for consistency.

**Tests**:
- Write file via MCP → list shows it immediately
- Delete file via MCP → list doesn't show it

---

#### Fix #10: Stash Name Validation (Security)
**File**: `src/core/manager.ts`

Names like `../etc` or `foo/bar` could cause path traversal. Add validation:

```typescript
const VALID_NAME = /^[a-zA-Z0-9][a-zA-Z0-9._-]*$/;

function validateStashName(name: string): void {
  if (!name || name.length > 64) {
    throw new Error("Stash name must be 1-64 characters");
  }
  if (!VALID_NAME.test(name)) {
    throw new Error(
      "Stash name must start with a letter or number and contain only "
      + "letters, numbers, dots, hyphens, or underscores"
    );
  }
}
```

**Tests**:
- Rejects `../etc`, `foo/bar`, `.hidden`, `-dash`, empty, 65+ chars
- Accepts `my-stash`, `stash.v2`, `Notes_2026`

---

#### Fix #7: Daemon MCP Race Condition (Correctness)
**File**: `src/daemon.ts`

Same `mcpServer` instance shared across concurrent requests. `connect()` overwrites transport.

**Fix**: Create fresh MCP server per request:

```typescript
app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,
  });
  const requestServer = createMcpServer(manager);  // Fresh per request
  res.on("close", () => transport.close());
  await requestServer.connect(transport);
  await transport.handleRequest(req, res, req.body);
});
```

**Tests**: Hard to unit test - may need integration test or manual verification.

---

#### Fix #2: Throttled Reload (Performance)
**File**: `src/core/manager.ts`, `src/mcp.ts`

Every MCP tool call runs `manager.reload()` - O(stashes × files) per request.

**Fix**: Add `reloadIfStale()` with 2s throttle:

```typescript
private lastReloadMs = 0;
private static RELOAD_INTERVAL_MS = 2000;

async reloadIfStale(): Promise<void> {
  const now = Date.now();
  if (now - this.lastReloadMs < StashManager.RELOAD_INTERVAL_MS) return;
  this.lastReloadMs = now;
  await this.reload();
}
```

**Tests**:
- First call reloads
- Immediate second call skips
- Call after 2s+ reloads again

---

### Medium Priority

#### Fix #1: Dirty Flag Generation Counter (Correctness)
**File**: `src/core/stash.ts`

`dirty` flag only clears in sync `.then()`. For local-only stashes (no provider), never clears.

**Fix**: Use generation counter, clear after save if no new writes:

```typescript
private saveGeneration = 0;

private scheduleBackgroundSave(): void {
  this.dirty = true;
  this.saveGeneration++;
  const generationAtSchedule = this.saveGeneration;
  
  this.savePromise = previousSave.then(() =>
    this.save()
      .then(() => {
        if (this.saveGeneration === generationAtSchedule) {
          this.dirty = false;
        }
      })
      .catch(...)
  );
  // Remove dirty=false from sync callback
}
```

**Tests**:
- Local-only stash: write → flush → isDirty() returns false
- Write A, write B quickly → dirty clears only after both saved

---

#### Fix #11: Config File Permissions (Security)
**File**: `src/core/config.ts`

GitHub token written with default umask, potentially readable by others.

**Fix**:
```typescript
await fs.mkdir(path.dirname(filePath), { recursive: true, mode: 0o700 });
await fs.writeFile(filePath, JSON.stringify(config, null, 2) + "\n", { mode: 0o600 });
```

**Tests**:
- After writeConfig, file has mode 0o600
- Directory has mode 0o700

---

#### Fix #6: setContent Uses Splice (Performance)
**File**: `src/core/file.ts`

`...content.split("")` creates N separate Automerge operations.

**Fix**: Use `splice` from `@automerge/automerge/next`:

```typescript
import { splice } from "@automerge/automerge/next";

export function setContent(doc, content): Automerge.Doc<TextFileDoc> {
  return Automerge.change(doc, (d) => {
    splice(d, ["content"], 0, d.content.length, content);
  });
}
```

**Tests**:
- Verify setContent still works correctly
- (Hard to test performance improvement directly)

**Risk**: Need to verify `@automerge/automerge/next` API compatibility.

---

### Low Priority

#### Fix #9: PID File for Daemon Stop (Portability)
**Files**: `src/daemon.ts`, `src/cli/commands/stop.ts`

Uses `lsof` which is macOS/Linux only.

**Fix**: Write PID file on start, read on stop.

**Tests**:
- Start creates PID file
- Stop reads PID, kills process, removes file
- Stale PID file (process dead) handled gracefully

---

#### Fix #4: Parallel GitHub Fetch (Performance)
**File**: `src/providers/github.ts`

Sequential fetches: structure → list docs → fetch each doc.

**Fix**: Use Trees API + parallel blob fetches.

**Tests**: Mock Octokit, verify parallel calls.

---

#### Fix #12: Remove Dead `inquirer` Dependency
**File**: `package.json`

`inquirer` listed but only `@inquirer/prompts` used.

**Fix**: `npm uninstall inquirer`

---

## Fixes Already Done / Not Needed

- **Fix #3**: Merge logic in provider → Already in main (`mergeWithRemote()`)
- **Fix #8**: promptSecret masking → Already uses `@inquirer/prompts` `password()`

---

## Implementation Order

1. ~~Fix #13 (reconciler scan on startup) - DONE~~
2. Fix #14 (MCP list/glob consistency) - correctness
3. Fix #10 (name validation) - security, simple
4. Fix #7 (daemon race) - correctness
5. Fix #2 (throttled reload) - performance
6. Fix #1 (dirty flag) - correctness
7. Fix #11 (config permissions) - security
8. Fix #12 (remove inquirer) - cleanup
9. Fix #6 (splice) - needs API verification
10. Fix #9 (PID file) - portability
11. Fix #4 (parallel fetch) - optimization

---

## After Fixes: Write SPEC.md

Document current architecture:
- Filesystem-first with reconciler
- Soft-delete tombstones
- Content-wins conflict resolution
- fetch/push provider interface
- knownPaths tracking
