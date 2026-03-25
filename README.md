# TypeScript Cache Action

A GitHub Action for incremental TypeScript type checking in CI by caching `.tsbuildinfo` files and restoring proper timestamps.

[![MIT License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## Overview

TypeScript's `tsc --build` mode uses file modification timestamps to determine what needs rebuilding. Git doesn't preserve timestamps, causing CI to rebuild everything unnecessarily. This action solves that by:

1. Restoring cached TypeScript output files (`.tsbuildinfo`, `.d.ts`)
2. Resetting all file timestamps to a stable baseline
3. Updating only changed files to current time
4. Enabling TypeScript to detect exactly what changed

**Result**: Only changed packages (and their dependents) are rebuilt, dramatically speeding up CI type checks.

## Quick Start

### Restore Cache (All Branches)

```yaml
- name: Restore TypeScript Cache
  uses: Attest/typescript-cache-action@main
  with:
    cache-base-key: ${{ runner.os }}
    cache-key: ${{ github.sha }}
    base-ref: ${{ github.event.repository.default_branch }}
    files: |
      **/*.tsbuildinfo
      **/types/**/*.d.ts

- name: Run Type Check
  run: pnpm exec tsc --build
```

### Save Cache (Main Branch Only)

```yaml
- name: Save TypeScript Cache
  if: github.ref == 'refs/heads/main'
  uses: Attest/typescript-cache-action/save@main
  with:
    cache-base-key: ${{ runner.os }}
    cache-key: ${{ github.sha }}
    files: |
      **/*.tsbuildinfo
      **/types/**/*.d.ts
```

## Complete Workflow Example

```yaml
name: Type Check

on:
  pull_request:
  push:
    branches: [main]

jobs:
  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install Dependencies
        run: pnpm install

      - name: Restore TypeScript Cache
        uses: Attest/typescript-cache-action@main
        with:
          cache-base-key: ${{ runner.os }}
          cache-key: ${{ github.sha }}
          base-ref: ${{ github.event.repository.default_branch }}
          files: |
            **/*.tsbuildinfo
            **/types/**/*.d.ts

      - name: Type Check
        run: pnpm exec tsc --build

      - name: Save TypeScript Cache
        if: github.ref == 'refs/heads/main'
        uses: Attest/typescript-cache-action/save@main
        with:
          cache-base-key: ${{ runner.os }}
          cache-key: ${{ github.sha }}
          files: |
            **/*.tsbuildinfo
            **/types/**/*.d.ts
```

## Inputs

### Restore Action

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `cache-base-key` | No | `''` | Base cache key for hierarchical cache lookup (e.g., `${{ runner.os }}`) |
| `cache-key` | No | `''` | Specific cache key (typically `${{ github.sha }}`) |
| `base-ref` | No | `main` | Base branch to compare against for change detection |
| `files` | **Yes** | - | Newline-separated glob patterns for TypeScript output files |

### Save Action

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `cache-base-key` | No | `''` | Base cache key (should match restore action) |
| `cache-key` | No | `''` | Specific cache key (should match restore action) |
| `files` | **Yes** | - | Newline-separated glob patterns (should match restore action) |

## How It Works

### The Problem

TypeScript's `tsc --build` mode compares file modification timestamps:
- If source files are **newer** than `.tsbuildinfo` → rebuild project
- If source files are **older** than `.tsbuildinfo` → skip rebuild

Git doesn't preserve timestamps. On checkout, all files get the **current** timestamp, making TypeScript think everything changed.

### The Solution

```
1. Restore Cache
   └─> Bring back .tsbuildinfo and .d.ts files from previous builds

2. Reset All Timestamps
   └─> Set all files to base branch commit timestamp (stable baseline)

3. Detect Changed Files
   └─> Compare current HEAD vs base branch using git

4. Update Changed Timestamps
   └─> Set only changed files to current timestamp (marking them as "new")

5. Run TypeScript
   └─> TypeScript sees: changed files = new, unchanged files = old
   └─> Only rebuilds changed packages + dependents
```

### Timestamp State

| Stage | Source Files | .tsbuildinfo | TypeScript Behavior |
|-------|-------------|--------------|---------------------|
| After Checkout | Current time | Missing | Rebuild everything |
| After Cache Restore | Current time | Base time | Rebuild everything |
| After Reset All | Base time | Base time | Skip everything |
| After Update Changed | Changed: Current<br>Unchanged: Base | Base time | Rebuild only changed |

## File Patterns

Choose patterns based on your TypeScript configuration:

### Common Patterns

```yaml
# Minimal (incremental build metadata only)
files: |
  **/*.tsbuildinfo

# With Generated Types
files: |
  **/*.tsbuildinfo
  **/types/**/*.d.ts

# With Custom Output Directory
files: |
  **/*.tsbuildinfo
  dist/**/*.d.ts
  build/**/*.d.ts

# Aggressive (all type definitions)
files: |
  **/*.tsbuildinfo
  **/*.d.ts
```

### Pattern Guidelines

- ✅ Cache build outputs (`.tsbuildinfo`, generated `.d.ts`)
- ❌ Don't cache source files (already in git)
- ❌ Don't cache `node_modules` (use separate cache)
- ⚠️  Larger patterns = slower cache restore/save

## Best Practices

### 1. Save Cache Only on Main Branch

```yaml
- if: github.ref == 'refs/heads/main'
  uses: Attest/typescript-cache-action/save@main
```

**Why**: Prevents cache pollution from experimental feature branches.

### 2. Use Consistent Keys

Restore and save actions should use **identical** inputs:
```yaml
# Both actions
cache-base-key: ${{ runner.os }}
cache-key: ${{ github.sha }}
files: |
  **/*.tsbuildinfo
```

### 3. Include OS in Cache Key

```yaml
cache-base-key: ${{ runner.os }}
```

**Why**: Build artifacts may differ between Linux, macOS, Windows.

### 4. Use SHA for Commit-Specific Cache

```yaml
cache-key: ${{ github.sha }}
```

**Why**: Each commit gets its own cache entry with fallback to prefix match.

### 5. Test File Patterns First

```yaml
- name: Verify Cache Contents
  run: |
    echo "Files to cache:"
    find . -name "*.tsbuildinfo" -o -name "types/**/*.d.ts"
```

## Performance

### Typical Speedup

| Scenario | Build Time | Speedup |
|----------|-----------|---------|
| No cache (baseline) | 100% | - |
| Full cache, no changes | ~5-10% | 10-20x faster |
| Full cache, small changes | ~10-30% | 3-10x faster |
| Full cache, large refactor | ~50-80% | 1.2-2x faster |

### Factors Affecting Performance

- **Cache Hit Rate**: Higher is better (aim for >80% on PRs)
- **Change Scope**: Fewer changed files = faster rebuilds
- **Dependency Graph**: Shallow dependencies = less cascading rebuilds
- **Project Structure**: Monorepos benefit more than single projects

## Troubleshooting

### Type Check Still Rebuilds Everything

**Possible Causes**:
1. Cache miss (no recent cache available)
2. File patterns don't match TypeScript output locations
3. Base branch not configured correctly

**Debug**:
```yaml
- name: Debug Cache
  run: |
    echo "Cache files found:"
    find . -name "*.tsbuildinfo"
    echo "Timestamps:"
    ls -lt **/*.tsbuildinfo | head -5
```

### Stale Cache Errors

**Symptom**: Type errors about missing files that were deleted

**Solution**: Clear cache and rebuild:
```bash
# Clear repository caches (requires admin)
gh api -X DELETE /repos/{owner}/{repo}/actions/caches
```

Or wait 7 days for automatic cache expiration.

### Cache Size Growing

**Solution**: Narrow file patterns to only necessary outputs:
```yaml
# Too broad (caches too much)
files: '**/*.d.ts'

# Better (only generated types)
files: '**/types/**/*.d.ts'
```

## Compatibility

| Platform | Status | Notes |
|----------|--------|-------|
| Ubuntu | ✅ Fully Supported | Primary platform, extensively tested |
| macOS | ✅ Fully Supported | Tested with latest runners |
| Windows | ⚠️ Untested | May require timestamp format adjustments |

### TypeScript Requirements

- ✅ `tsc --build` mode (project references)
- ❌ Plain `tsc` (doesn't use incremental metadata)

### Monorepo Tools

Compatible with:
- pnpm workspaces
- Nx
- Turborepo (alongside its own cache)
- Lerna
- Yarn workspaces

## Advanced Usage

### Multiple Cache Strategies

```yaml
# OS + Node version specific cache
cache-base-key: ${{ runner.os }}-node${{ matrix.node-version }}
cache-key: ${{ github.sha }}
```

### Custom Base Branch

```yaml
# For release branches
base-ref: release/v2.0
```

### Conditional Cache Restore

```yaml
# Skip cache on force rebuild
- if: "!contains(github.event.head_commit.message, '[no-cache]')"
  uses: Attest/typescript-cache-action@main
```

## Contributing

We use this action across Attest projects and welcome contributions!

### Development

```bash
# Test locally by referencing local path in workflow
uses: ./.github/actions/typescript-cache-action
```

### Reporting Issues

- **Bugs**: Open an issue with workflow logs and repository structure
- **Feature Requests**: Describe use case and expected behavior
- **Security**: See [security guidelines](.agents/rules/security.md)

## Documentation

- **[AGENTS.md](AGENTS.md)** - Comprehensive technical reference for AI agents
- **[ARCHITECTURE.md](docs/ARCHITECTURE.md)** - System design and diagrams
- **[Code Style](.agents/rules/code-style.md)** - YAML and shell conventions
- **[Testing](.agents/rules/testing.md)** - Testing strategies and best practices
- **[Security](.agents/rules/security.md)** - Security considerations and guidelines

## License

MIT License - see [LICENSE](LICENSE) for details

Copyright (c) 2025 Attest Technologies Limited

## Acknowledgments

- Built on top of GitHub's official [actions/cache](https://github.com/actions/cache)
- Uses [tj-actions/changed-files](https://github.com/tj-actions/changed-files) for change detection
- Maintained by [@Attest/frontend](https://github.com/orgs/Attest/teams/frontend)

---

**Status**: Production • **Maintained By**: @Attest/frontend • **License**: MIT
