<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.7

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sphinx-notes--pages/3.7** was hardened automatically. 2 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

action.yml has uses: references pinned to tags or branches instead of 40-char commit SHAs: actions/checkout@v6 (twice), actions/setup-python@v6, astral-sh/setup-uv@v10.3.0, actions/cache@v5, sphinx-doc/github-problem-matcher@master, actions/configure-pages@v6, actions/upload-pages-artifact@v5.0.0, actions/deploy-pages@v5.

Locations:

- `action.yml:66`
- `action.yml:71`
- `action.yml:75`
- `action.yml:83`
- `action.yml:89`
- `action.yml:98`
- `action.yml:116`
- `action.yml:134`
- `action.yml:141`

### script-injection (severity: high)

Sub-rule (a): a GitHub Actions expression is placed directly in the run: block of the 'Build documentation' step: `run: ${{ github.action_path }}/main.sh`. Pass it through an env var and quote it, e.g. "$GITHUB_ACTION_PATH/main.sh".

Locations:

- `action.yml:102`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection

**Notes:**

Pinned actions/checkout (x2), setup-python, astral-sh/setup-uv, cache, sphinx-doc/github-problem-matcher, configure-pages, upload-pages-artifact and deploy-pages to full commit SHAs, each with a tag comment. Replaced `run: ${{ github.action_path }}/main.sh` with the quoted "$GITHUB_ACTION_PATH/main.sh" (shell: bash).

### Iteration 2

**Fixes applied:** unpinned-uses

**Notes:**

Pinned the three remaining tag references in action.yml to full commit SHAs, keeping the tag as a trailing comment:
- actions/setup-python@ece7cb06caefa5fff74198d8649806c4678c61a1 # v6
- actions/configure-pages@45bfe0192ca1faeb007ade9deae92b16b8254a0d # v6
- actions/upload-pages-artifact@fc324d3547104276b827a68afc52ff2a11cc49c9 # v5.0.0

The file was already broken when I opened it. The INPUT_SPHINX_BUILD_OPTIONS line in the build step's env had text from a configure-pages pin pasted into the middle of it, which made the YAML invalid. I rebuilt that part as 'INPUT_SPHINX_BUILD_OPTIONS: ${{ inputs.sphinx_build_options }}' followed by a configure-pages step. The step name 'Setup Pages' is my guess, because the original name was lost in the damage. Running several edits on the file at once caused the same kind of damage here: one of my first three edits didn't stick and I had to redo it. I made the remaining edits one at a time and checked the whole file afterwards.

