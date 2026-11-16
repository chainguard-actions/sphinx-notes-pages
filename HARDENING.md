<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.7

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sphinx-notes--pages/3.7** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

action.yml references actions by mutable tag or branch, not by a full 40-character commit SHA: actions/checkout@v6 (twice), actions/setup-python@v6, astral-sh/setup-uv@v10.3.0, actions/cache@v5, sphinx-doc/github-problem-matcher@master, actions/configure-pages@v6, actions/upload-pages-artifact@v5.0.0, actions/deploy-pages@v5.

Locations:

- `action.yml:65`
- `action.yml:70`
- `action.yml:74`
- `action.yml:82`
- `action.yml:88`
- `action.yml:98`
- `action.yml:116`
- `action.yml:133`
- `action.yml:140`

### script-injection (severity: high)

Rule (a): the 'Build documentation' step puts a GitHub expression directly inside a run: block: `run: ${{ github.action_path }}/main.sh`. Pass it through env (or use "$GITHUB_ACTION_PATH/main.sh") instead.

Locations:

- `action.yml:102`

### script-injection (severity: high)

Rule (b): main.sh, which the build step runs, expands workflow-controlled input env vars without quotes: `pkg_install -U sphinx==$INPUT_SPHINX_VERSION` (line 45), `pkg_install .[$INPUT_PYPROJECT_EXTRAS]` (line 74), and `$sphinx_build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir"` (line 103). These come from inputs.sphinx_version, inputs.pyproject_extras and inputs.sphinx_build_options. None of them is an array assignment with a shellcheck SC2086/SC2206 directive, so the exemption does not apply.

Locations:

- `main.sh:45`
- `main.sh:74`
- `main.sh:103`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 4cc4ee8..3e91e7e 100644
--- a/action.yml
+++ b/action.yml
@@ -61,17 +61,34 @@ outputs:
 runs:
   using: "composite"
   steps:
     - name: Checkout
       uses: actions/checkout@v6
       if: ${{ inputs.checkout == 'true' && inputs.cache == 'true' }}
       with:
         fetch-depth: 0 # Required by sphinxnotes-incrbuild
     - name: Checkout
       uses: actions/checkout@v6
       if: ${{ inputs.checkout == 'true' && inputs.cache == 'false' }}
 
     - name: Setup python
       uses: actions/setup-python@v6
       with:
         python-version: ${{ inputs.python_version }}
         # Empty string disables caching; pip cache is pointless on the uv path
@@ -79,13 +96,13 @@ runs:
 
     - name: Setup uv
       # setup-uv only publishes exact version tags, no major tags like v10
       uses: astral-sh/setup-uv@v10.3.0
       if: ${{ inputs.pypi_installer == 'uv' }}
       with:
         enable-cache: ${{ inputs.cache }}
 
     - name: Restore cache
       uses: actions/cache@v5
       if: ${{ inputs.cache == 'true' }}
       with:
         path: /tmp/sphinxnotes-incrbuild
@@ -95,11 +112,11 @@ runs:
           sphinxnotes-pages-${{ runner.os }}
 
     - name: Enable github problem matcher
       uses: sphinx-doc/github-problem-matcher@master
 
     - id: build
       name: Build documentation
-      run: ${{ github.action_path }}/main.sh
+      run: '"$GITHUB_ACTION_PATH/main.sh"'
       shell: bash
       env:
         # See https://github.com/actions/runner/issues/665
@@ -113,7 +130,7 @@ runs:
         INPUT_SPHINX_BUILD_OPTIONS: ${{ inputs.sphinx_build_options }}
 
     - name: Setup Pages
       uses: actions/configure-pages@v6
       if: ${{ inputs.publish == 'true' }}
 
     - name: Fix file permissions
@@ -130,12 +147,12 @@ runs:
 
     - name: Upload artifact
       # https://github.com/actions/upload-pages-artifact/issues/142
       uses: actions/upload-pages-artifact@v5.0.0
       if: ${{ inputs.publish == 'true' }}
       with:
         path: ${{ steps.build.outputs.artifact }}
       
     - id: deployment
       name: Deploy to GitHub Pages
       uses: actions/deploy-pages@v5
       if: ${{ inputs.publish == 'true' }}

```

Locations:

- `action.yml`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, prior-fix

**Notes:**

Pinned all 9 uses: references in action.yml to full commit SHAs, keeping the tag as a trailing comment. Changed the build step's run: to '"$GITHUB_ACTION_PATH/main.sh"', which re-applies the previous harden's change. In main.sh, quoted "sphinx==$INPUT_SPHINX_VERSION" and ".[$INPUT_PYPROJECT_EXTRAS]". sphinx_build_options holds a list of options, so it now goes into an array with an unquoted assignment and a shellcheck SC2206 directive. That splits and globs the value the same way upstream's unquoted expansion did, and the array is passed as "${sphinx_build_options[@]}". Nothing was run or tested; these are edits only.

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

main.sh now double-quotes every variable it expands from workflow input in an echo: the repository and documentation paths (L15-16), the requirements path (L62, L66), pyproject extras (L72) and pyproject group (L82). I also quoted the Action and Workspace echo lines, the 'Installing sphinx via' line and the $GITHUB_OUTPUT redirect target, which weren't in the finding. Nothing was run to test this. The command arguments that were already quoted are unchanged. The sphinx_build_options array still splits and globs the input the same way upstream did.

