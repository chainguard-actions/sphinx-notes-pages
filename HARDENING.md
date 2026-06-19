<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.6

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **sphinx-notes--pages/3.6** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

action.yml contains 9 `uses:` references pinned to mutable version tags or branch names instead of immutable 40-character SHA commit hashes. This exposes the action to supply-chain attacks if any upstream actions are compromised or their tags are moved. Failing references: actions/checkout@v6 (x2), actions/setup-python@v6 (x2), actions/cache@v5, sphinx-doc/github-problem-matcher@master, actions/configure-pages@v6, actions/upload-pages-artifact@v5.0.0, actions/deploy-pages@v5.

Locations:

- `action.yml:54`
- `action.yml:57`
- `action.yml:60`
- `action.yml:64`
- `action.yml:68`
- `action.yml:76`
- `action.yml:93`
- `action.yml:105`
- `action.yml:110`

### script-injection (severity: high)

Sub-rule (a): A ${{ }} expression is interpolated directly inside a run: shell command string. The offending line in action.yml is: `run: ${{ github.action_path }}/main.sh`. Any ${{ ... }} expression directly in a run: block is a script-injection risk because the value is substituted into the shell command string before the shell parses it. This should be replaced with the pre-set environment variable $GITHUB_ACTION_PATH instead.

Locations:

- `action.yml:79`

### script-injection (severity: high)

Sub-rule (b): In main.sh, the env var $INPUT_SPHINX_BUILD_OPTIONS (sourced from inputs.sphinx_build_options, a user-controlled input set via the env: block in action.yml) is expanded unquoted in the shell command: `if ! $sphinx_build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir";`. An unquoted shell variable expansion allows an attacker to inject shell metacharacters (;, |, &, $(...), etc.) via the sphinx_build_options input, enabling arbitrary command execution. It should be double-quoted.

Locations:

- `main.sh:63`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all 9 unpinned-uses in action.yml: actions/checkout@v6 (×2) → SHA df4cb1c, actions/setup-python@v6 (×2) → SHA a309ff8, actions/cache@v5 → SHA 27d5ce7, sphinx-doc/github-problem-matcher@master → SHA 1f74d65, actions/configure-pages@v6 → SHA 45bfe01, actions/upload-pages-artifact@v5.0.0 → SHA fc324d3, actions/deploy-pages@v5 → SHA cd2ce8f. Fixed script-injection in action.yml line 79 by replacing ${{ github.action_path }}/main.sh with $GITHUB_ACTION_PATH/main.sh. Fixed script-injection in main.sh line 63 by replacing unquoted $INPUT_SPHINX_BUILD_OPTIONS with a bash array (IFS=' ' read -r -a sphinx_build_options <<< "$INPUT_SPHINX_BUILD_OPTIONS") and expanding it as "${sphinx_build_options[@]}" to prevent shell metacharacter injection while preserving multi-option support.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed two unquoted shell variable expansions in main.sh that allowed shell injection via caller-controlled inputs:
1. Line 27: Changed `pip3 install -U sphinx==$INPUT_SPHINX_VERSION` to `pip3 install -U "sphinx==$INPUT_SPHINX_VERSION"` — the version string is now double-quoted, preventing injection via semicolons, pipes, or command substitution.
2. Line 54: Changed `pip3 install .[$INPUT_PYPROJECT_EXTRAS]` to `pip3 install ".[${INPUT_PYPROJECT_EXTRAS}]"` — the extras specifier is now double-quoted with braces around the variable name, preventing similar injection attacks.

