# Code Style Guidelines

## Action Definition (YAML)

### File Naming
- Main action: `action.yml` (not `action.yaml`)
- Subdirectory actions: `{subdir}/action.yml`

### Structure
```yaml
name: 'Action Name'
description: 'Clear, concise description'
branding:
  icon: 'github-octicons-name'
  color: 'color-name'

inputs:
  input-name:
    description: 'What this input does'
    required: true|false
    default: 'default-value'  # if not required

runs:
  using: 'composite'
  steps:
    - name: descriptive step name
      shell: bash
      run: |
        command here
```

### Input Naming
- Use kebab-case: `cache-base-key`, `base-ref`
- Be descriptive but concise
- Document clearly what each input does

### Step Naming
- Use descriptive, action-oriented names
- Good: "restore tsc build cache", "fetch base branch"
- Avoid: "step 1", "cache", "files"

## Shell Scripts

### Commands
- Use `shell: bash` for all shell steps
- Prefer built-in commands (`find`, `touch`, `git`) over custom scripts
- Use `-c` flag with `touch` to avoid errors on missing files

### Formatting
- For multi-line commands, use YAML's `|` or `>` operators
- For single commands, inline is fine
- Keep commands readable - break long lines with `\`

### Example
```yaml
- name: restore timestamps of all files to base branch commit
  shell: bash
  run: |
    find . -type f \
      -exec touch -c -m -t $(git log -1 --format=%cd --date=format:%y%m%d%H%M.%S ${{ inputs.base-ref }}) {} +
```

## GitHub Actions Expressions

### Inputs
- Reference with `${{ inputs.input-name }}`
- Always use inputs for configurable values
- Don't hardcode values that might change per project

### Steps Output
- Use step IDs for referencing outputs: `steps.step-id.outputs.output-name`
- Check conditionals: `steps.step-id.outputs.any_changed == 'true'`

### Context Variables
- Use GitHub context thoughtfully: `github.sha`, `github.ref`, `runner.os`
- Document expected context in README examples

## Conditional Logic

### When to Use
- `if:` conditions for optional steps
- Common: `if: steps.changed-files.outputs.any_changed == 'true'`
- Recommend workflow-level conditions in documentation (e.g., only save on main)

### Boolean Checks
```yaml
# Check string equality
if: steps.step-id.outputs.value == 'true'

# Check existence
if: steps.step-id.outputs.value != ''

# Workflow context
if: github.ref == 'refs/heads/main'
```

## Documentation in Code

### Action Metadata
- `name`: User-facing action name (shows in marketplace)
- `description`: One-sentence summary of what the action does
- Input `description`: Clear explanation of purpose and format

### Comments
- Minimal inline comments in YAML (should be self-documenting)
- Use step names to explain intent
- Complex commands deserve comments above them

## Error Handling

### Defensive Coding
- Use `touch -c` to avoid errors on missing files
- Use `git fetch` before referencing remote branches
- Check step outputs before using them (`if:` conditions)

### Graceful Degradation
- Cache restore should not fail the workflow if cache misses
- Use `restore-keys` for fallback cache matches

## Performance

### Minimize Steps
- Combine related commands in single steps when logical
- Don't over-split simple operations
- Each step has overhead in action runner

### Efficient Commands
- Use `find ... -exec ... +` instead of piping to `xargs`
- Minimize git operations (fetch once, use multiple times)

## Dependencies

### Action Versions
- Pin to major version with `@v4` (allows minor/patch updates)
- Document breaking changes if updating major versions
- Track upstream action changes for security

### External Actions
- Prefer official GitHub actions (`actions/*`)
- Use well-maintained community actions (`tj-actions/*`)
- Document why each external action is needed

## Consistency

### Across Actions
- Use same input names across restore/save actions
- Match parameter descriptions
- Keep branding consistent

### With Ecosystem
- Follow GitHub Actions conventions
- Match patterns from popular actions (e.g., `actions/cache`)
- Use standard input names when applicable (`path`, `key`)
