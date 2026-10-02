<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.6

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sphinx-notes--pages/3.6** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

All 'uses:' references in action.yml use mutable tags or version strings instead of pinned 40-character SHA commit digests. This exposes the action to supply-chain attacks if any upstream action is compromised or its tag is moved. Failing references: actions/checkout@v6 (×2), actions/setup-python@v6 (×2), actions/cache@v5, sphinx-doc/github-problem-matcher@master, actions/configure-pages@v6, actions/upload-pages-artifact@v5.0.0, actions/deploy-pages@v5.

Locations:

- `action.yml:53`
- `action.yml:57`
- `action.yml:61`
- `action.yml:66`
- `action.yml:71`
- `action.yml:77`
- `action.yml:84`
- `action.yml:91`
- `action.yml:97`

### script-injection (severity: high)

Sub-rule (a): The 'Build documentation' run: block directly interpolates the GitHub Actions expression ${{ github.action_path }} into the shell command string: `run: ${{ github.action_path }}/main.sh`. Any ${{ ... }} expression inside a run: block is subject to YAML template substitution before the shell processes it, making it a script-injection risk. The value should instead be accessed via the $GITHUB_ACTION_PATH environment variable, which is already available in composite action steps.

Locations:

- `action.yml:83`

### script-injection (severity: high)

Sub-rule (b): In main.sh, two env vars sourced from user-controlled inputs are expanded unquoted in shell commands, allowing shell metacharacter injection:
1. `$INPUT_SPHINX_BUILD_OPTIONS` (from inputs.sphinx_build_options) is unquoted in: `if ! $sphinx_build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir"` — an attacker can inject arbitrary shell commands via this input.
2. `$INPUT_SPHINX_VERSION` (from inputs.sphinx_version) is unquoted in: `pip3 install -U sphinx==$INPUT_SPHINX_VERSION` — shell metacharacters in the version string are not quoted.
Both variables should be double-quoted: "$INPUT_SPHINX_BUILD_OPTIONS" and "$INPUT_SPHINX_VERSION".

Locations:

- `main.sh:63`
- `main.sh:24`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Fixed all findings in action.yml and main.sh:

1. unpinned-uses: Pinned all 9 'uses:' references to full 40-char SHA digests with original tags as comments. Used lookup_action_sha to resolve each: actions/checkout@v6→d23441a4, actions/setup-python@v6→ece7cb06, actions/cache@v5→caa29612, sphinx-doc/github-problem-matcher@master→1f74d659, actions/configure-pages@v6→45bfe019, actions/upload-pages-artifact@v5.0.0→fc324d35, actions/deploy-pages@v5→368f8252.

2. script-injection (a): Replaced `run: ${{ github.action_path }}/main.sh` with `run: "$GITHUB_ACTION_PATH/main.sh"` — uses the pre-set environment variable instead of a YAML-interpolated expression.

3. script-injection (b): In main.sh, quoted $INPUT_SPHINX_VERSION as "sphinx==$INPUT_SPHINX_VERSION". For $INPUT_SPHINX_BUILD_OPTIONS (a list of flags), used the guarded xargs+read-loop pattern to tokenize into a bash array (sphinx_build_opts) and expanded it as "${sphinx_build_opts[@]}" — preserves argument boundaries and prevents shell metacharacter injection.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all unquoted shell variable expansions of user-controlled inputs in main.sh:
1. Lines 8-9: Quoted `$INPUT_REPOSITORY_PATH` and `$INPUT_DOCUMENTATION_PATH` in variable assignments to prevent word splitting.
2. Lines 42, 45: Quoted `$INPUT_REQUIREMENTS_PATH` in echo statements.
3. Line 50: Quoted `$INPUT_PYPROJECT_EXTRAS` in echo statement.
4. Line 52 (critical): Changed `pip3 install .[$INPUT_PYPROJECT_EXTRAS]` to `pip3 install ".[${INPUT_PYPROJECT_EXTRAS}]"` — the unquoted form allowed injection of shell metacharacters (`;`, `|`, `$(...)`) into the pip3 command. The entire argument is now double-quoted so the shell treats it as a single string, preventing command injection.

