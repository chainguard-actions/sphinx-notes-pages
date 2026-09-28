<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sphinx-notes--pages/3.4** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All 7 `uses:` references in action.yml are pinned to mutable version tags or branch names rather than immutable 40-character SHA digests. This exposes the action to supply-chain attacks if any upstream action is compromised or its tag is moved. Failing references:
- `actions/checkout@v4` (appears twice)
- `actions/setup-python@v5` (appears twice)
- `actions/cache@v4`
- `sphinx-doc/github-problem-matcher@master`
- `actions/configure-pages@v4`
- `actions/upload-pages-artifact@v3`
- `actions/deploy-pages@v4`

Locations:

- `action.yml:57`
- `action.yml:60`
- `action.yml:63`
- `action.yml:67`
- `action.yml:71`
- `action.yml:79`
- `action.yml:96`
- `action.yml:107`
- `action.yml:111`

### script-injection (severity: high)

Sub-rule (a): The 'Build documentation' step directly interpolates the GitHub Actions expression `${{ github.action_path }}` inside the `run:` shell command string (`run: ${{ github.action_path }}/main.sh`). Any `${{ ... }}` expression interpolated directly into a `run:` block undergoes YAML template substitution before the shell sees it, making it a script-injection risk. The safe alternative is to use the `$GITHUB_ACTION_PATH` environment variable instead.

Locations:

- `action.yml:82`

### script-injection (severity: high)

Sub-rule (b): In main.sh, the variable `$INPUT_SPHINX_BUILD_OPTIONS` (sourced from `inputs.sphinx_build_options`, an untrusted caller-controlled input) is used **unquoted** in the sphinx-build invocation: `sphinx-build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir"`. An unquoted shell expansion allows an attacker to inject shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) via the `sphinx_build_options` input, achieving arbitrary command execution.

Locations:

- `main.sh:80`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all findings in action.yml and main.sh:

1. **unpinned-uses**: Pinned all 7 action references (9 occurrences) to full SHA digests:
   - actions/checkout@v4 → @11d5960a326750d5838078e36cf38b85af677262 (×2)
   - actions/setup-python@v5 → @a26af69be951a213d495a4c3e4e4022e16d87065 (×2)
   - actions/cache@v4 → @0057852bfaa89a56745cba8c7296529d2fc39830
   - sphinx-doc/github-problem-matcher@master → @1f74d6599f4a5e89a20d3c99aab4e6a70f7bda0f
   - actions/configure-pages@v4 → @1f0c5cde4bc74cd7e1254d0cb4de8d49e9068c7d
   - actions/upload-pages-artifact@v3 → @56afc609e74202658d3ffba0e8f6dda462b719fa
   - actions/deploy-pages@v4 → @d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e

2. **script-injection (a)**: Replaced `run: ${{ github.action_path }}/main.sh` with `run: "$GITHUB_ACTION_PATH/main.sh"` to use the safe environment variable instead of a template expression.

3. **script-injection (b)**: Replaced unquoted `$INPUT_SPHINX_BUILD_OPTIONS` in main.sh with a properly guarded xargs-based array tokenization pattern (`sphinx_build_opts` array) that preserves argument boundaries and prevents shell metacharacter injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two script injection vulnerabilities in hardened/action/main.sh:
1. Line 28: Changed `pip3 install -U sphinx==$INPUT_SPHINX_VERSION` to `pip3 install -U "sphinx==$INPUT_SPHINX_VERSION"` — quoting prevents shell metacharacter injection from the sphinx_version input.
2. Line 55: Changed `pip3 install .[$INPUT_PYPROJECT_EXTRAS]` to `pip3 install ".[${INPUT_PYPROJECT_EXTRAS}]"` — quoting prevents shell metacharacter injection from the pyproject_extras input.

### Iteration 3

**Fixes applied:** script-injection

**Notes:**

Fixed all 6 unquoted shell variable expansion violations in main.sh:
1. Line 8: Quoted `$GITHUB_WORKSPACE/$INPUT_REPOSITORY_PATH` in repo_dir assignment
2. Line 9: Quoted `$repo_dir/$INPUT_DOCUMENTATION_PATH` in doc_dir assignment
3. Line 38: Quoted the echo string containing `$INPUT_REQUIREMENTS_PATH`
4. Line 40: Quoted the echo string containing `$INPUT_REQUIREMENTS_PATH`
5. Line 46: Quoted the echo string containing `$INPUT_PYPROJECT_EXTRAS`
6. Line 75: Double-quoted both `$git_restore_mtime` (command) and `$repo_dir` (argument)

