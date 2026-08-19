<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.6

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sphinx-notes--pages/3.6** was hardened automatically. 3 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple `uses:` references in action.yml and .github/workflows/pages.yml use mutable version tags instead of pinned 40-character SHA commit digests, making the action vulnerable to supply-chain attacks if any upstream action is compromised or its tag is moved.

Failing references in action.yml:
- `actions/checkout@v6` (×2, lines 57 and 62)
- `actions/setup-python@v6` (×2, lines 66 and 72)
- `actions/cache@v5` (line 78)
- `sphinx-doc/github-problem-matcher@master` (line 88) — uses a branch name
- `actions/configure-pages@v6` (line 104)
- `actions/upload-pages-artifact@v5.0.0` (line 121)
- `actions/deploy-pages@v5` (line 128)

Failing reference in .github/workflows/pages.yml:
- `sphinx-notes/pages@v3` (line 22)

Locations:

- `action.yml:57`
- `action.yml:62`
- `action.yml:66`
- `action.yml:72`
- `action.yml:78`
- `action.yml:88`
- `action.yml:104`
- `action.yml:121`
- `action.yml:128`
- `.github/workflows/pages.yml:22`

### script-injection (severity: high)

Two script-injection issues found:

**(a) Direct expression interpolation in run: block** — action.yml line 92 uses `${{ github.action_path }}` directly inside a `run:` shell command string:
```yaml
run: ${{ github.action_path }}/main.sh
```
Any `${{ ... }}` expression interpolated directly into a `run:` block is a script-injection risk because the value is substituted by the YAML template engine before the shell ever sees it, bypassing shell quoting.

**(b) Unquoted shell variable expansions of workflow-controllable data in main.sh** — Two env vars sourced from `inputs.*` are expanded without double-quoting:

1. Line 28: `pip3 install -U sphinx==$INPUT_SPHINX_VERSION` — `$INPUT_SPHINX_VERSION` (from `inputs.sphinx_version`) is unquoted, allowing shell metacharacter injection.

2. Line 63: `$sphinx_build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir"` — `$INPUT_SPHINX_BUILD_OPTIONS` (from `inputs.sphinx_build_options`) is unquoted, allowing an attacker to inject arbitrary additional arguments or shell metacharacters (e.g., `;`, `|`, `$(...)`).

Locations:

- `action.yml:92`
- `main.sh:28`
- `main.sh:63`

### permissions (severity: medium)

The workflow file .github/workflows/pages.yml has no top-level `permissions:` key. While the single job `pages` does define job-level permissions (`pages: write` and `id-token: write`), these are narrowly scoped only to what that job needs. However, the absence of a top-level `permissions: {}` means the default GitHub token permissions (which can be broad, including `contents: write` on some repository configurations) apply to any future jobs added to this workflow that do not explicitly declare their own permissions block. Best practice is to add a top-level `permissions: {}` to deny all permissions by default and grant only what each job needs.

Locations:

- `.github/workflows/pages.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, permissions

**Notes:**

Fixed all three findings:

1. **unpinned-uses**: Pinned all 9 unpinned action references to full 40-char SHA digests in action.yml (actions/checkout×2, actions/setup-python×2, actions/cache, sphinx-doc/github-problem-matcher, actions/configure-pages, actions/upload-pages-artifact, actions/deploy-pages) and in .github/workflows/pages.yml (sphinx-notes/pages). Original tags preserved as comments.

2. **script-injection**: (a) Moved `${{ github.action_path }}` from the `run:` string into the `env:` block as `ACTION_PATH`, with the run command using `"$ACTION_PATH/main.sh"`. (b) Quoted `$INPUT_SPHINX_VERSION` in the pip install command. (c) Used `read -ra sphinx_build_options <<< "$INPUT_SPHINX_BUILD_OPTIONS"` to safely split build options into an array before passing to sphinx-build.

3. **permissions**: Added `permissions: {}` top-level block to .github/workflows/pages.yml to deny all permissions by default; job-level permissions (pages: write, id-token: write) remain for the pages job.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed all five unquoted shell variable expansions of user-controlled inputs in main.sh: (1) repo_dir assignment now double-quotes $INPUT_REPOSITORY_PATH, (2) doc_dir assignment now double-quotes $INPUT_DOCUMENTATION_PATH, (3) echo for requirements group now double-quotes the entire string including $INPUT_REQUIREMENTS_PATH, (4) echo for pyproject group now double-quotes the entire string including $INPUT_PYPROJECT_EXTRAS, (5) most critically, `pip3 install .[$INPUT_PYPROJECT_EXTRAS]` is now `pip3 install ".[$INPUT_PYPROJECT_EXTRAS]"` preventing shell metacharacter injection via the pyproject_extras input.

