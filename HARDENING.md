<!-- markdownlint-disable -->

# Hardening Report: sphinx-notes--pages/3.6

> This file was generated automatically by the hardening agent.

**Policy SHA:** `374911c4be584df24e6da57af2d990570ef0f859`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **sphinx-notes--pages/3.6** was hardened automatically. 5 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

action.yml has composite-step uses: references that point to a mutable tag or branch instead of a full 40-character commit SHA: actions/checkout@v6 (twice), actions/setup-python@v6 (twice), actions/cache@v5, sphinx-doc/github-problem-matcher@master, actions/configure-pages@v6, actions/upload-pages-artifact@v5.0.0, actions/deploy-pages@v5.

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

### script-injection (severity: high)

Rule (a): the 'Build documentation' step puts an expression directly in the run: string: `run: ${{ github.action_path }}/main.sh`. Pass the path through the environment instead, e.g. `run: "$GITHUB_ACTION_PATH/main.sh"`.

Locations:

- `action.yml:92`

### script-injection (severity: high)

Rule (b): main.sh is the script the composite step runs, and it expands workflow-controlled input env vars without quotes: `pip3 install -U sphinx==$INPUT_SPHINX_VERSION`, `pip3 install .[$INPUT_PYPROJECT_EXTRAS]`, `$sphinx_build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir"`, and the derived paths `repo_dir=$GITHUB_WORKSPACE/$INPUT_REPOSITORY_PATH` / `doc_dir=$repo_dir/$INPUT_DOCUMENTATION_PATH` used unquoted in echo. INPUT_SPHINX_BUILD_OPTIONS is meant to hold several options, but it has no shellcheck SC2086/SC2206 array-assignment hardening, so the exemption does not apply.

Locations:

- `main.sh:28`
- `main.sh:57`
- `main.sh:73`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed action.yml without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/action.yml b/action.yml
index 85d6575..6efb9d8 100644
--- a/action.yml
+++ b/action.yml
@@ -49,33 +49,47 @@ outputs:
   artifact:
     description: 'Directory where artifact (HTML documentation) is stored, user can use it to deploy GitHub Pages manually'
     value: ${{ steps.build.outputs.artifact }}
-
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
-
     - name: Setup python
       uses: actions/setup-python@v6
       if: ${{ inputs.cache == 'true' }}
       with:
         python-version: ${{ inputs.python_version }}
         cache: 'pip'
     - name: Setup python
       uses: actions/setup-python@v6
       if: ${{ inputs.cache == 'false' }}
       with:
         python-version: ${{ inputs.python_version }}
-
     - name: Restore cache
       uses: actions/cache@v5
       if: ${{ inputs.cache == 'true' }}
       with:
         path: /tmp/sphinxnotes-incrbuild
@@ -83,13 +97,11 @@ runs:
         key: sphinxnotes-pages-${{ runner.os }}-${{ github.run_id }}
         restore-keys: |
           sphinxnotes-pages-${{ runner.os }}
-
     - name: Enable github problem matcher
       uses: sphinx-doc/github-problem-matcher@master
-
     - id: build
       name: Build documentation
-      run: ${{ github.action_path }}/main.sh
+      run: "$GITHUB_ACTION_PATH/main.sh"
       shell: bash
       env:
         # See https://github.com/actions/runner/issues/665
@@ -99,11 +111,9 @@ runs:
         INPUT_SPHINX_VERSION: ${{ inputs.sphinx_version }}
         INPUT_CACHE: ${{ inputs.cache }}
         INPUT_SPHINX_BUILD_OPTIONS: ${{ inputs.sphinx_build_options }}
-
     - name: Setup Pages
       uses: actions/configure-pages@v6
       if: ${{ inputs.publish == 'true' }}
-
     - name: Fix file permissions
       shell: sh
       if: runner.os == 'Linux'
@@ -115,15 +125,13 @@ runs:
         done
       env:
         INPUT_PATH: ${{ steps.build.outputs.artifact }}
-
     - name: Upload artifact
       # https://github.com/actions/upload-pages-artifact/issues/142
       uses: actions/upload-pages-artifact@v5.0.0
       if: ${{ inputs.publish == 'true' }}
       with:
         path: ${{ steps.build.outputs.artifact }}
-      
     - id: deployment
       name: Deploy to GitHub Pages
       uses: actions/deploy-pages@v5
       if: ${{ inputs.publish == 'true' }}

```

Locations:

- `action.yml`

### prior-fix (severity: high)

Carried forward from this version's previous harden, which fixed it; fix it again unless the code no longer has the problem. The previous harden changed main.sh without recording a finding for it, and this analysis reported nothing there. Re-apply its security changes unless the code no longer needs them. The previous patch:
```diff
diff --git a/main.sh b/main.sh
index 65e2eb7..7e00c63 100755
--- a/main.sh
+++ b/main.sh
@@ -5,8 +5,8 @@ set -e
 
 echo ::group:: Initialize various paths
 
-repo_dir=$GITHUB_WORKSPACE/$INPUT_REPOSITORY_PATH
-doc_dir=$repo_dir/$INPUT_DOCUMENTATION_PATH
+repo_dir="$GITHUB_WORKSPACE/$INPUT_REPOSITORY_PATH"
+doc_dir="$repo_dir/$INPUT_DOCUMENTATION_PATH"
 # https://stackoverflow.com/a/4774063/4799273
 action_dir=$GITHUB_ACTION_PATH
 
@@ -25,7 +25,7 @@ echo Installing sphinx via pip
 if [ -z "$INPUT_SPHINX_VERSION" ] ; then
     pip3 install -U sphinx
 else
-    pip3 install -U sphinx==$INPUT_SPHINX_VERSION
+    pip3 install -U "sphinx==$INPUT_SPHINX_VERSION"
 fi
 
 echo Adding ~/.local/bin to system path
@@ -42,19 +42,19 @@ pip3 install -U sphinxnotes-incrbuild>=1.0
 echo ::endgroup::
 
 if [ ! -z "$INPUT_REQUIREMENTS_PATH" ] ; then
-    echo ::group:: Installing dependencies declared by $INPUT_REQUIREMENTS_PATH
+    echo "::group:: Installing dependencies declared by $INPUT_REQUIREMENTS_PATH"
     if [ -f "$INPUT_REQUIREMENTS_PATH" ]; then
         pip3 install -r "$INPUT_REQUIREMENTS_PATH"
     else
-        echo No $INPUT_REQUIREMENTS_PATH found, skipped
+        echo "No $INPUT_REQUIREMENTS_PATH found, skipped"
     fi
     echo ::endgroup::
 fi
 
 if [ ! -z "$INPUT_PYPROJECT_EXTRAS" ] ; then
-    echo ::group:: Installing dependencies declared by pyproject.toml[$INPUT_PYPROJECT_EXTRAS]
+    echo "::group:: Installing dependencies declared by pyproject.toml[$INPUT_PYPROJECT_EXTRAS]"
     if [ -f "pyproject.toml" ]; then
-        pip3 install .[$INPUT_PYPROJECT_EXTRAS]
+        pip3 install ".[${INPUT_PYPROJECT_EXTRAS}]"
     else
         echo No pyproject.toml found, skipped
     fi
@@ -70,7 +70,12 @@ if [ "$INPUT_CACHE" == "true" ]; then
 else
     sphinx_build=sphinx-build
 fi
-if ! $sphinx_build -b html $INPUT_SPHINX_BUILD_OPTIONS "$doc_dir" "$build_dir"; then
+sphinx_build_opts=()
+if [ -n "$INPUT_SPHINX_BUILD_OPTIONS" ]; then
+    while IFS= read -r -d '' t; do sphinx_build_opts+=("$t"); done \
+        < <(printf '%s' "$INPUT_SPHINX_BUILD_OPTIONS" | xargs printf '%s\0')
+fi
+if ! $sphinx_build -b html "${sphinx_build_opts[@]}" "$doc_dir" "$build_dir"; then
     for l in $(find /tmp -name 'sphinx-err*.log' 2>/dev/null); do
         # Replace "\n" to "%0A" for supporting multiline text in the error message.
         # https://github.com/actions/toolkit/issues/193#issuecomment-605394935

```

Locations:

- `main.sh`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, script-injection, prior-fix

**Notes:**

Pinned all 9 composite uses: references in action.yml to full commit SHAs (looked up), keeping the tag as a trailing comment. The 'Build documentation' step no longer has an inline expression: it now runs "$GITHUB_ACTION_PATH/main.sh", quoted. In main.sh I quoted the derived paths (repo_dir, doc_dir, action_dir) and the echo lines that use them, plus the sphinx version spec ("sphinx==$INPUT_SPHINX_VERSION"), the extras spec (".[${INPUT_PYPROJECT_EXTRAS}]"), the requirements messages, build_dir, $l and $GITHUB_OUTPUT. Upstream expanded sphinx_build_options unquoted, so I now split it into an array with the same unquoted assignment (sphinx_build_opts=($INPUT_SPHINX_BUILD_OPTIONS), with shellcheck SC2086/SC2206 disabled). That keeps upstream's whitespace split and globbing and evaluates nothing. I chose this over the previous harden's xargs split, which handled quotes differently from upstream (it honoured quotes upstream treated as plain text). The array is then passed as "${sphinx_build_opts[@]}". I didn't run the action or any tests.

