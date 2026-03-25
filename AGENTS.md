# AGENTS.md

## Project Overview

**typescript-cache-action** is a GitHub Action that enables incremental TypeScript type checking in CI environments by managing `.tsbuildinfo` file caching and restoring proper file timestamps.

### Core Problem
TypeScript's `tsc --build` mode relies on file modification timestamps to determine which packages need rebuilding. Git doesn't preserve timestamps, causing CI to rebuild everything unnecessarily.

### Solution
1. Restore cached TypeScript output files (`.tsbuildinfo`, `.d.ts`)
2. Reset all file timestamps to the base branch commit timestamp
3. Update timestamps for git-changed files to current time
4. Run `tsc --build` - only rebuilds changed packages
5. Save updated cache for future runs

## Repository Structure

```
typescript-cache-action/
├── action.yml              # Main restore action definition
├── save/
│   └── action.yml          # Cache save action definition
├── README.md               # User-facing documentation
├── LICENSE                 # MIT License
└── .github/
    └── CODEOWNERS          # Code ownership (@Attest/frontend)
```

## Architecture

This is a **composite GitHub Action** (not a TypeScript/JavaScript action). It uses shell commands and existing GitHub Actions to orchestrate the caching workflow.

### Main Action (action.yml)
**Purpose**: Restore cache and prepare timestamps before type checking

**Inputs**:
- `cache-base-key`: Base key for cache lookups (e.g., `${{ runner.os }}`)
- `cache-key`: Specific cache key (e.g., `${{ github.sha }}`)
- `base-ref`: Base branch for timestamp restoration (default: `main`)
- `files`: Glob patterns for TypeScript output files (required)

**Steps**:
1. Fetch base-ref branch
2. Restore cache using `actions/cache/restore@v4`
3. Touch all files with base branch commit timestamp
4. Detect changed files using `tj-actions/changed-files@v45`
5. Touch changed files with current timestamp

### Save Action (save/action.yml)
**Purpose**: Save TypeScript build cache after successful type check

**Inputs**:
- `cache-base-key`: Base key for cache storage
- `cache-key`: Specific cache key
- `files`: Glob patterns for TypeScript output files (required)

**Steps**:
1. Save cache using `actions/cache/save@v4`

## Usage Patterns

### Typical Workflow

```yaml
# Restore cache before type checking (all branches)
- uses: Attest/typescript-cache-action@main
  with:
    cache-base-key: ${{ runner.os }}
    cache-key: ${{ github.sha }}
    base-ref: ${{ github.event.repository.default_branch }}
    files: |
      **/types/**/*.d.ts
      **/*.tsbuildinfo

# Run type check
- run: pnpm exec tsc --build

# Save cache (only on main branch)
- if: github.ref == 'refs/heads/main'
  uses: Attest/typescript-cache-action/save@main
  with:
    cache-base-key: ${{ runner.os }}
    cache-key: ${{ github.sha }}
    files: |
      **/types/**/*.d.ts
      **/*.tsbuildinfo
```

### Best Practices
1. **Save cache only on default branch** - Prevents cache pollution from feature branches
2. **Use consistent file patterns** - Ensure restore and save use identical `files` globs
3. **Include OS in cache key** - Different OS may produce different build artifacts
4. **Use SHA for cache key** - Ensures each commit has unique cache entry

## Key Dependencies

### External Actions
- `actions/cache/restore@v4` - GitHub's cache restore action
- `actions/cache/save@v4` - GitHub's cache save action
- `tj-actions/changed-files@v45` - Detects files changed between commits

### Shell Commands
- `git fetch` - Retrieves base branch reference
- `find` with `touch` - Resets file timestamps to base branch time
- `touch` - Updates timestamps for changed files

## Technical Details

### Timestamp Strategy

**Goal**: Make TypeScript's incremental build detection work correctly

**Base State** (after base branch timestamp restore):
- All source files: base commit timestamp
- All `.tsbuildinfo` files: base commit timestamp (from cache)

**After Changed File Update**:
- Unchanged source files: base commit timestamp (older than `.tsbuildinfo`)
- Changed source files: current timestamp (newer than `.tsbuildinfo`)
- `.tsbuildinfo` files: base commit timestamp

**TypeScript Behavior**:
- Compares source file timestamps vs `.tsbuildinfo` timestamps
- Rebuilds packages where source is newer than `.tsbuildinfo`
- Only changed packages (and dependents) are rebuilt

### Cache Strategy

**Cache Key Hierarchy**:
1. `{cache-base-key}-{cache-key}` - Exact match (e.g., `Linux-abc123`)
2. `{cache-base-key}-` - Prefix match (e.g., `Linux-*`)

**Benefits**:
- Exact match: Fastest, restores exact build state
- Prefix match: Falls back to most recent build on same OS
- Graceful degradation if exact cache unavailable

### File Patterns

Common patterns for TypeScript projects:
```
**/*.tsbuildinfo           # TypeScript incremental build files
**/types/**/*.d.ts         # Generated type definitions
**/*.d.ts                  # All type definition files (broader)
dist/**/*.d.ts             # Distribution type definitions
build/**/*.d.ts            # Build output type definitions
```

## Constraints & Limitations

1. **Composite Actions Only**: No compiled TypeScript/JavaScript - only shell commands and action composition
2. **Git Required**: Depends on git for change detection and branch operations
3. **Linux/macOS Focus**: `touch` and `find` commands assume Unix-like systems
4. **Cache Size Limits**: GitHub Actions cache has 10GB limit per repository
5. **Base Branch Required**: Must have fetched base branch for timestamp restoration

## Maintenance Guidelines

### When Updating Actions
- Test with various monorepo structures (Nx, Turborepo, pnpm workspaces)
- Verify timestamp restoration on both Linux and macOS runners
- Check cache hit rates in real workflows
- Ensure changed file detection works with merge commits

### Monitoring
- Cache hit rates (should be high on PR builds)
- Type check duration (should decrease with cache)
- Cache size (monitor for excessive growth)

### Versioning
- Use semantic versioning for releases
- Maintain `@main` for latest stable
- Test breaking changes on feature branches before merging

## Common Issues

### Issue: Type check still rebuilds everything
**Causes**:
- Cache miss (no recent cache available)
- File patterns don't match actual output locations
- Base branch not fetched correctly

**Debugging**:
1. Check cache hit/miss in action logs
2. Verify file patterns match actual TypeScript output
3. Ensure base-ref points to valid branch

### Issue: Stale cache causes type errors
**Cause**: Cached `.tsbuildinfo` references files that no longer exist

**Solution**: Clear cache or let it expire naturally (GitHub cache has 7-day retention)

### Issue: Cache size growing too large
**Causes**:
- Too many files in `files` pattern
- Including source files instead of just build outputs
- Multiple cache entries per commit

**Solution**:
- Narrow file patterns to only necessary build outputs
- Use cache retention policies
- Save cache only on main branch

## Related Documentation

- [TypeScript Project References](https://www.typescriptlang.org/docs/handbook/project-references.html)
- [TypeScript Build Mode](https://www.typescriptlang.org/docs/handbook/project-references.html#build-mode-for-typescript)
- [GitHub Actions Cache](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows)
- [Composite Actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)

---

**Ownership**: @Attest/frontend
**License**: MIT
**Status**: Production (In use across Attest projects)
