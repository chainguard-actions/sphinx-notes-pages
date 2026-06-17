<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.4

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `1`

Action **sphinx-notes--pages/3.4** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references in action.yml are pinned to mutable tags or branch names instead of full 40-character commit SHAs. This exposes the action to supply-chain attacks if any upstream action is compromised or its tag is moved. Failing references:
- `actions/checkout@v4` (appears twice)
- `actions/setup-python@v5` (appears twice)
- `actions/cache@v4`
- `sphinx-doc/github-problem-matcher@master` (branch name — highest risk)
- `actions/configure-pages@v4`
- `actions/upload-pages-artifact@v3`
- `actions/deploy-pages@v4`

Locations:

- `action.yml:52`
- `action.yml:55`
- `action.yml:58`
- `action.yml:63`
- `action.yml:68`
- `action.yml:76`
- `action.yml:80`
- `action.yml:88`
- `action.yml:93`

### script-injection (severity: high)

Sub-rule (a): A `${{ }}` expression is directly interpolated inside a `run:` shell command string. The line `run: ${{ github.action_path }}/main.sh` in action.yml embeds the `github.action_path` context value directly into the shell command before the shell ever sees it. Although `github.action_path` is typically controlled by GitHub, any `${{ ... }}` expression in a `run:` block is a script-injection risk per the check rules, as YAML template substitution occurs before shell quoting.

Locations:

- `action.yml:82`

### script-injection (severity: high)

Sub-rule (b): Unquoted shell variable expansions of untrusted (caller-controlled) inputs in main.sh:

1. `$INPUT_SPHINX_BUILD_OPTIONS` is expanded unquoted in the sphinx-build command: `sphinx-build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir"`. This env var is set from `inputs.sphinx_build_options` (caller-controlled). An attacker can inject shell metacharacters (`;`, `|`, `&`, `$(...)`, etc.) via this input.

2. `$INPUT_SPHINX_VERSION` is expanded unquoted in `pip3 install -U sphinx==$INPUT_SPHINX_VERSION`. This env var is set from `inputs.sphinx_version` (caller-controlled) and is unquoted, allowing shell metacharacter injection.

Locations:

- `main.sh:24`
- `main.sh:80`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all findings in action.yml and main.sh:

1. unpinned-uses: Pinned all 9 `uses:` references to full 40-char commit SHAs with tag comments preserved:
   - actions/checkout@v4 → @34e114876b0b11c390a56381ad16ebd13914f8d5 (×2)
   - actions/setup-python@v5 → @a26af69be951a213d495a4c3e4e4022e16d87065 (×2)
   - actions/cache@v4 → @0057852bfaa89a56745cba8c7296529d2fc39830
   - sphinx-doc/github-problem-matcher@master → @1f74d6599f4a5e89a20d3c99aab4e6a70f7bda0f
   - actions/configure-pages@v4 → @1f0c5cde4bc74cd7e1254d0cb4de8d49e9068c7d
   - actions/upload-pages-artifact@v3 → @56afc609e74202658d3ffba0e8f6dda462b719fa
   - actions/deploy-pages@v4 → @d6db90164ac5ed86f2b6aed7e0febac5b3c0c03e

2. script-injection (sub-rule a): Moved `github.action_path` out of the `run:` string into the step's `env:` block as `ACTION_PATH`; run now uses `"$ACTION_PATH/main.sh"`.

3. script-injection (sub-rule b): In main.sh, quoted `$INPUT_SPHINX_VERSION` as `"sphinx==$INPUT_SPHINX_VERSION"` and replaced unquoted `$INPUT_SPHINX_BUILD_OPTIONS` with a bash array (`IFS=' ' read -ra sphinx_build_opts <<< "$INPUT_SPHINX_BUILD_OPTIONS"`) expanded as `"${sphinx_build_opts[@]}"`.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed script injection vulnerability in main.sh line 51: changed `pip3 install .[$INPUT_PYPROJECT_EXTRAS]` to `pip3 install ".[$INPUT_PYPROJECT_EXTRAS]"`. Double-quoting the entire argument (including the surrounding brackets) prevents shell metacharacter interpretation of attacker-controlled input while preserving the correct pip extras specifier syntax.

