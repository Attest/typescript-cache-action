# Testing Guidelines

## Testing Strategy

Since this is a composite GitHub Action (not a TypeScript/JavaScript action), traditional unit tests don't apply. Testing focuses on integration and real-world workflows.

## Manual Testing

### Test Environments

#### Required Test Cases
1. **Fresh Cache** - First run with no cache
2. **Cache Hit** - Subsequent run with exact cache match
3. **Cache Miss (Partial)** - Run with prefix cache match only
4. **Changed Files** - Run with modified source files
5. **Unchanged Files** - Run with no changes (full cache benefit)

#### Test Projects
- **Monorepo**: Multiple TypeScript projects with project references
- **Single Project**: Simple TypeScript project
- **Large Project**: Project with many files (test performance)

### Test Workflow

Create a `.github/workflows/test-typescript-cache.yml`:

```yaml
name: Test TypeScript Cache Action

on:
  pull_request:
  push:
    branches: [main]

jobs:
  test-cache:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - run: pnpm install

      # Test restore action
      - name: Restore TypeScript Cache
        uses: ./  # Test local action
        with:
          cache-base-key: ${{ runner.os }}-test
          cache-key: ${{ github.sha }}
          base-ref: main
          files: |
            **/*.tsbuildinfo
            **/types/**/*.d.ts

      # Run type check and measure time
      - name: Type Check (First Run)
        run: time pnpm exec tsc --build

      # Test save action
      - name: Save TypeScript Cache
        if: github.ref == 'refs/heads/main'
        uses: ./save
        with:
          cache-base-key: ${{ runner.os }}-test
          cache-key: ${{ github.sha }}
          files: |
            **/*.tsbuildinfo
            **/types/**/*.d.ts

      # Test cache hit on same SHA
      - name: Clean TypeScript outputs
        run: pnpm exec tsc --build --clean

      - name: Restore TypeScript Cache (Second Run)
        uses: ./
        with:
          cache-base-key: ${{ runner.os }}-test
          cache-key: ${{ github.sha }}
          base-ref: main
          files: |
            **/*.tsbuildinfo
            **/types/**/*.d.ts

      - name: Type Check (Cached Run)
        run: time pnpm exec tsc --build
```

### Validation Checklist

After running test workflow, verify:

- [ ] Cache restore completes without errors
- [ ] Changed files detected correctly
- [ ] Timestamps set appropriately (check with `ls -lt`)
- [ ] Type check completes successfully
- [ ] Cache save completes (on main branch only)
- [ ] Subsequent run is faster (cache hit)
- [ ] Only changed packages rebuild on PR branches

## Integration Testing

### Cross-Platform Testing

Test on multiple runners:
```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
```

Verify:
- `touch` command works correctly
- `find` command behavior consistent
- Git operations succeed
- Timestamp format compatible

### Monorepo Testing

Test with different monorepo tools:
- **pnpm workspaces**: Test workspace dependencies
- **Nx**: Test affected project detection
- **Turborepo**: Test turbo cache interaction
- **Lerna**: Test multi-package builds

### Edge Cases

#### Empty Cache
- **Scenario**: First run or cache expired
- **Expected**: Action completes, type check runs fully, cache saved
- **Verify**: No errors, subsequent runs faster

#### No Changed Files
- **Scenario**: Re-run on same commit
- **Expected**: All files have base branch timestamp
- **Verify**: Type check completes instantly (no rebuild)

#### All Files Changed
- **Scenario**: Large refactor or branch divergence
- **Expected**: All packages rebuild
- **Verify**: Behavior same as no cache, but cache still saved

#### Missing Base Branch
- **Scenario**: `base-ref` doesn't exist
- **Expected**: Action fails gracefully with clear error
- **Verify**: Error message helpful, workflow can catch and handle

#### File Pattern Mismatch
- **Scenario**: `files` glob doesn't match any TypeScript outputs
- **Expected**: Cache empty but action succeeds
- **Verify**: No errors, type check runs fully

## Performance Testing

### Metrics to Track

1. **Cache Restore Time**: Should be < 10s for most projects
2. **Timestamp Update Time**: Should be < 5s for most projects
3. **Type Check Time (No Cache)**: Baseline measurement
4. **Type Check Time (Full Cache)**: Should be ~90% faster
5. **Type Check Time (Partial Cache)**: Should be 50-80% faster
6. **Cache Save Time**: Should be < 10s for most projects

### Benchmark Workflow

```yaml
- name: Benchmark No Cache
  run: |
    tsc --build --clean
    time tsc --build

- name: Restore Cache
  uses: ./
  with:
    cache-base-key: ${{ runner.os }}
    cache-key: ${{ github.sha }}
    base-ref: main
    files: '**/*.tsbuildinfo'

- name: Benchmark With Cache
  run: time tsc --build
```

Compare times and ensure cache provides meaningful speedup.

## Debugging

### Enable Debug Logging

Set in workflow or repository secrets:
```yaml
env:
  ACTIONS_STEP_DEBUG: true
```

### Inspect Timestamps

Add debugging step:
```yaml
- name: Debug Timestamps
  run: |
    echo "Base branch commit time:"
    git log -1 --format=%cd --date=format:%y%m%d%H%M.%S ${{ inputs.base-ref }}
    echo "Sample file timestamps:"
    ls -lt **/*.tsbuildinfo | head -5
    ls -lt src/**/*.ts | head -5
```

### Verify Cache Contents

```yaml
- name: Verify Cache Hit
  if: steps.cache-restore.outputs.cache-hit == 'true'
  run: |
    echo "Cache hit! Verifying contents..."
    find . -name "*.tsbuildinfo" -type f
```

### Test Changed File Detection

```yaml
- name: Debug Changed Files
  run: |
    echo "Changed files:"
    echo "${{ steps.changed-files.outputs.all_changed_files }}"
    echo "Any changed: ${{ steps.changed-files.outputs.any_changed }}"
```

## Regression Testing

### Before Releasing

1. Test with real Attest projects
2. Verify cache hit rates unchanged
3. Check type check durations don't regress
4. Test on both Linux and macOS runners
5. Verify with fresh cache (cache cleared)

### Breaking Changes

If changing:
- Input names → Major version bump
- Cache key format → Document migration path
- File patterns → Test with existing caches
- Timestamp logic → Extensive testing required

## Continuous Validation

### Monitoring in Production

Track in real workflows:
- Cache hit rate (aim for >80% on PR builds)
- Type check duration trends
- Cache size growth
- Failure rates

### Feedback Loop

Collect feedback from:
- Attest frontend team
- GitHub Actions logs
- Developer reports
- Performance metrics

## Documentation Testing

### Verify Examples Work

Copy-paste examples from README into test workflow:
- Ensure syntax correct
- Verify inputs valid
- Check outputs as expected
- Confirm best practices followed

### Test Documentation Scenarios

For each usage example in README:
1. Create minimal reproduction
2. Run through workflow
3. Verify expected behavior
4. Document any surprises
