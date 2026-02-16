# Plan: Import existing repo files on create

## Goal
When `stash create` points at a GitHub repo that has existing files (but no `.stash/`), prompt the user to either adopt those files as the stash content, or abort.

## Current behavior
1. `create` calls `provider.fetch()` to get `.stash/` docs
2. If `remoteDocs.size > 0` → error "use connect"
3. If `remoteDocs.size === 0` → call `provider.create()` to init `.stash/`

## Problem
A repo with regular files but no `.stash/` passes the check, but we'd be ignoring those files. User expectation: those files should become the stash.

## Proposed behavior
After seeing `remoteDocs.size === 0`:
1. Check if repo has any other files (outside `.stash/`)
2. If yes, prompt:
   ```
   This repository has 47 existing files.
   Convert to stash and import them? [Y/n]
   - Y: Import all files, initialize .stash
   - n: Abort (don't create partial stash)
   ```
3. If user confirms, fetch all file contents and write to stash
4. If user aborts, exit without creating anything

## Implementation

### 1. Add `listFiles()` to GitHubProvider
**File:** `src/providers/github.ts`

Add method to list all files in repo (excluding `.stash/`):
```typescript
async listFiles(): Promise<string[]> {
  // Use GitHub tree API to get all files
  // Filter out .stash/ paths
  // Return relative paths
}
```

### 2. Add `fetchFile(path)` to GitHubProvider
**File:** `src/providers/github.ts`

Fetch a single file's content:
```typescript
async fetchFile(path: string): Promise<Buffer> {
  // Get file content from GitHub
}
```

### 3. Update create command
**File:** `src/cli/commands/create.ts`

After `remoteDocs.size === 0` check:
```typescript
// Check for existing files
const existingFiles = await provider.listFiles();
if (existingFiles.length > 0) {
  const confirm = await promptChoice(
    `This repository has ${existingFiles.length} existing files.\nConvert to stash and import them?`,
    ["Yes, import files", "No, abort"]
  );
  
  if (confirm === "No, abort") {
    console.log("Aborted. Repository unchanged.");
    process.exit(0);
  }
  
  // Will import after stash creation below
}

// Initialize .stash structure
await provider.create();

// Create local stash
const stash = await manager.create(...);

// Import remote files if any
if (existingFiles.length > 0) {
  console.log(`Importing ${existingFiles.length} files...`);
  for (const filePath of existingFiles) {
    const content = await provider.fetchFile(filePath);
    if (isTextFile(filePath, content)) {
      stash.write(filePath, content.toString('utf-8'));
    } else {
      stash.writeBinary(filePath, ...);
    }
  }
  await stash.flush();
}
```

### 4. Update SyncProvider interface
**File:** `src/providers/types.ts`

Add optional methods:
```typescript
interface SyncProvider {
  // ... existing
  listFiles?(): Promise<string[]>;
  fetchFile?(path: string): Promise<Buffer>;
}
```

## Testing

1. **Empty repo** - works as before
2. **Repo with .stash/** - errors "use connect" as before  
3. **Repo with files, user confirms** - imports all files
4. **Repo with files, user aborts** - exits cleanly, no changes
5. **Repo with mixed text/binary** - handles both correctly

## Edge cases

- **Large repos**: May need progress indicator for many files
- **Binary files**: Detect and use writeBinary()
- **Nested directories**: Preserve full paths
- **Hidden files**: Import .gitignore etc (but not .git/)
