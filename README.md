# TypeScript Cache Action

This action allows caching TypeScript build information and types for incremental typecheck runs within github actions.

This is still WIP.

## Documentation

```yaml
name: Caching TypeScript build

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # Any other steps (ie generation of files)

      - name: Restore Typescript Cache
        uses: Attest/typescript-cache-action@main
        with:
          # base cache key
          cache-base-key: ${{ runner.os }}
          # cache key
          cache-key: ${{ github.sha }}
          # base ref to restore timestamps to
          base-ref: ${{ github.event.repository.default_branch }}
          # output files to restore cache for
          files: |
            **/types/**/*.d.ts
            **/*.tsbuildinfo
 
      - name: run tsc
        run: pnpm exec tsc --build
      
      # Save TypeScript cache (recommend only on base branch workflow) 
      - name: Save Typescript Cache
        if: ${{ github.workflow == 'main' }}
        uses: Attest/typescript-cache-action/save@main
        with:
          # base cache key
          cache-base-key: ${{ runner.os }}
          # cache key
          cache-key: ${{ github.sha }}
          # output files to restore cache for
          files: |
            **/types/**/*.d.ts
            **/*.tsbuildinfo

```

### CI problems

TypeScript build mode (project references) checks what packages to build by comparing the modified timestamps of source and `.tsbuildinfo` files. If the src files are newer than the `.tsbuildinfo` file then it will detect this project and its dependencies need to be built. This works perfectly in local environments however in CI, this raises issues since git is decentralised and thus file meta information like timestamps are not stored. This makes sense for git as files being touched should not cause git to detect changes, only content changes should be checked in.

To improve the CI type checking stages the `.tsbuildinfo` and source files need to have their timestamps restored in a fashion that allows `tsc --build` mode to detect what to build, otherwise all packages will be rebuilt which is slower.

#### Restore cache of TS output files

Firstly this action restore the latest cache of the typescript output files (`files` input) from the default branch (`base-ref` input). This gives us the most recent and most stable cache.

#### Restore timestamps to base ref

At this stage, the cached timestamps are when the cache was created and the source files are when checked out. The modified timestamps of all files are restored to the base branch head commit timestamp. This ensures that the timestamps for all files are stable and are before any committed changes.

### Restore timestamps of changed files in git
For any changed files detected by git the modified timestamps are restored to current timestamp.

This gives us a state of:

- ts build modified timestamps: base commit timestamp
- unchanged source modified timestamps: base commit timestamp
- changed source modified timestamps: current timestamp

### Run type check

As the state of the modified timestamps are corrected typescript can now run and detect changed packages. It will now only rebuild these packages and their dependents.

Run typecheck at this stage

### Save cache of ts build files

After the type check as been run the ts build output has been updated. If on the default branch we save this cache to speed up further builds.