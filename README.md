# clang-tidy pull request comments

[![clang-tidy-8 support]](https://releases.llvm.org/8.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)
[![clang-tidy-9 support]](https://releases.llvm.org/9.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)
[![clang-tidy-10 support]](https://releases.llvm.org/10.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)
[![clang-tidy-11 support]](https://releases.llvm.org/11.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)
[![clang-tidy-12 support]](https://releases.llvm.org/12.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)
[![clang-tidy-13 support]](https://releases.llvm.org/13.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)
[![clang-tidy-14 support]](https://releases.llvm.org/14.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)
[![clang-tidy-15 support]](https://releases.llvm.org/15.0.0/tools/clang/tools/extra/docs/clang-tidy/index.html)

A GitHub Action to post `clang-tidy` warnings and suggestions as review comments on your pull request.

![action preview](https://i.imgur.com/lQiFdT9.png)

* [What](#what)
  * [Supported clang-tidy versions](#supported-clang-tidy-versions)
* [How](#how)
  * [Basic configuration example](#basic-configuration-example)
  * [Triggering this Action manually](#triggering-this-action-manually)
  * [Using this Action to safely perform analysis of pull requests from forks](#using-this-action-to-safely-perform-analysis-of-pull-requests-from-forks)
* [Who's using this action?](#whos-using-this-action)

## What

`platisd/clang-tidy-pr-comments` is a GitHub Action that utilizes the *exported fixes* of
`clang-tidy` for your C++ project and posts them as **code review comments** in the related **pull request**.

If `clang-tidy` has a concrete recommendation on how you should modify your code to fix the issue that's detected,
then it will be presented as a *suggested change* that can be committed directly. Alternatively,
the offending line will be highlighted along with a description of the warning.

The GitHub Action can be configured to *request changes* if `clang-tidy` warnings are found or merely
leave a comment without blocking the pull request from being merged. It should fail only if it has been
misconfigured by you, due to a bug (please contact me if that's the case) or the GitHub API acting up.

Please note the following:

* It will **not** run `clang-tidy` for you. You are responsible for doing that and then supply the Action with
  the path to your generated report (see [examples](#how) below). You can generate a
  YAML report that includes *fixes* for a pull request using the following methods:

  * Using the `run-clang-tidy` utility script with the `-export-fixes` argument. This script usually
    comes with the `clang-tidy` packages. You can use it to run checks *for the entire codebase* of a
    project at once.

  * Using the `clang-tidy-diff` utility script with the `-export-fixes` argument. This script also usually
    comes with the `clang-tidy` packages, and and it can be used to run checks only for code fragments that
    *have been changed* in a specific pull request.

  * Alternatively, you may use `--export-fixes` with `clang-tidy` itself in your own script.

  In any case, specify the path where you would like the report to be exported. The very same path should
  be supplied to this Action.

* It will *only* comment on files and lines changed in the pull request. This is due to GitHub not allowing
  comments on other files outside the pull request `diff`.
  This means that there may be more warnings in your project. Make sure you fix them *before* starting to
  use this Action to ensure new warnings will not be introduced in the future.

* This Action *respects* existing comments and doesn't repeat the same warnings for the same line (no spam).

* This Action allows analysis to be performed *separately* from the posting of the analysis results (using
  separate workflows with different privileges), which
  [allows you to safely analyze pull requests from forks](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)
  (see [example](#using-this-action-to-safely-perform-analysis-of-pull-requests-from-forks) below).

### Supported clang-tidy versions

YAML files containing generated fixes by the following `clang-tidy` versions are currently supported:
* `clang-tidy-8`
* `clang-tidy-9`
* `clang-tidy-10`
* `clang-tidy-11`
* `clang-tidy-12`
* `clang-tidy-13`
* `clang-tidy-14`
* `clang-tidy-15`

## How

Since this action comments on files changed in pull requests, naturally, it can be only run
on `pull_request` events. That being said, if it happens to be triggered in a different context,
e.g. a `push` event, it will **not** run and fail *softly* by returning a *success* code.

### Basic configuration example

A basic configuration for the `platisd/clang-tidy-pr-comments` action (for a `CMake`-based project
using the `clang-tidy-diff` script) can be seen below:

```yaml
name: Static analysis

on: pull_request

jobs:
  clang-tidy:
    runs-on: ubuntu-22.04
    - uses: actions/checkout@v4
      with:
        ref: ${{ github.event.pull_request.head.sha }}
        fetch-depth: 0
    - name: Fetch base branch
      run: |
        git remote add upstream "https://github.com/${{ github.event.pull_request.base.repo.full_name }}"
        git fetch --no-tags --no-recurse-submodules upstream "${{ github.event.pull_request.base.ref }}"
    - name: Install clang-tidy
      run: |
        sudo apt-get update
        sudo apt-get install -y clang-tidy
    - name: Prepare compile_commands.json
      run: |
        cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
    - name: Create results directory
      run: |
        mkdir clang-tidy-result
    - name: Analyze
      run: |
        git diff -U0 "$(git merge-base HEAD "upstream/${{ github.event.pull_request.base.ref }}")" | clang-tidy-diff -p1 -path build -export-fixes clang-tidy-result/fixes.yml
    - name: Run clang-tidy-pr-comments action
      uses: platisd/clang-tidy-pr-comments@v1
      with:
        # The GitHub token (or a personal access token)
        github_token: ${{ secrets.GITHUB_TOKEN }}
        # The path to the clang-tidy fixes generated previously
        clang_tidy_fixes: clang-tidy-result/fixes.yml
        # Optionally set to true if you want the Action to request
        # changes in case warnings are found
        request_changes: true
        # Optionally set the number of comments per review
        # to avoid GitHub API timeouts for heavily loaded
        # pull requests
        suggestions_per_comment: 10
```

### Triggering this Action manually

If you want to trigger this Action manually, i.e. by leaving a comment with a particular *keyword*
in the pull request, then you can try the following:

```yaml
name: Static analysis

# Don't trigger it on pull_request events but issue_comment instead
on: issue_comment

jobs:
  clang-tidy:
    # Trigger the job only when someone comments: run_clang_tidy
    if: ${{ github.event.issue.pull_request && contains(github.event.comment.body, 'run_clang_tidy') }}
    runs-on: ubuntu-22.04
    steps:
    - uses: actions/checkout@v4
      with:
        ref: ${{ github.event.pull_request.head.sha }}
        fetch-depth: 0
    - name: Fetch base branch
      run: |
        git remote add upstream "https://github.com/${{ github.event.pull_request.base.repo.full_name }}"
        git fetch --no-tags --no-recurse-submodules upstream "${{ github.event.pull_request.base.ref }}"
    - name: Install clang-tidy
      run: |
        sudo apt-get update
        sudo apt-get install -y clang-tidy
    - name: Prepare compile_commands.json
      run: |
        cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
    - name: Create results directory
      run: |
        mkdir clang-tidy-result
    - name: Analyze
      run: |
        git diff -U0 "$(git merge-base HEAD "upstream/${{ github.event.pull_request.base.ref }}")" | clang-tidy-diff -p1 -path build -export-fixes clang-tidy-result/fixes.yml
    - name: Run clang-tidy-pr-comments action
      uses: platisd/clang-tidy-pr-comments@v1
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        clang_tidy_fixes: clang-tidy-result/fixes.yml
```

### Using this Action to safely perform analysis of pull requests from forks

If you want to trigger this Action using the `workflow_run` event to run analysis on pull requests
from forks in a
[secure manner](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/),
then you can use the following combination of workflows:

```yaml
# Insecure workflow with limited permissions that should provide analysis results through an artifact
name: Static analysis

on: pull_request

jobs:
  clang-tidy:
    runs-on: ubuntu-22.04
    steps:
    - uses: actions/checkout@v4
      with:
        ref: ${{ github.event.pull_request.head.sha }}
        fetch-depth: 0
    - name: Fetch base branch
      run: |
        git remote add upstream "https://github.com/${{ github.event.pull_request.base.repo.full_name }}"
        git fetch --no-tags --no-recurse-submodules upstream "${{ github.event.pull_request.base.ref }}"
    - name: Install clang-tidy
      run: |
        sudo apt-get update
        sudo apt-get install -y clang-tidy
    - name: Prepare compile_commands.json
      run: |
        cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
    - name: Create results directory
      run: |
        mkdir clang-tidy-result
    - name: Analyze
      run: |
        git diff -U0 "$(git merge-base HEAD "upstream/${{ github.event.pull_request.base.ref }}")" | clang-tidy-diff -p1 -path build -export-fixes clang-tidy-result/fixes.yml
    - name: Save PR metadata
      run: |
        echo "${{ github.event.number }}" > clang-tidy-result/pr-id.txt
        echo "${{ github.event.pull_request.head.repo.full_name }}" > clang-tidy-result/pr-head-repo.txt
        echo "${{ github.event.pull_request.head.sha }}" > clang-tidy-result/pr-head-sha.txt
    - uses: actions/upload-artifact@v3
      with:
        name: clang-tidy-result
        path: clang-tidy-result/
```

```yaml
# Secure workflow with access to repository secrets and GitHub token for posting analysis results
name: Post the static analysis results

on:
  workflow_run:
    workflows: [ "Static analysis" ]
    types: [ completed ]

jobs:
  clang-tidy-results:
    # Trigger the job only if the previous (insecure) workflow completed successfully
    if: ${{ github.event.workflow_run.event == 'pull_request' && github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-22.04
    steps:
    - name: Download analysis results
      uses: actions/github-script@v6
      with:
        script: |
          let artifacts = await github.rest.actions.listWorkflowRunArtifacts({
              owner: context.repo.owner,
              repo: context.repo.repo,
              run_id: ${{ github.event.workflow_run.id }},
          });
          let matchArtifact = artifacts.data.artifacts.filter((artifact) => {
              return artifact.name == "clang-tidy-result"
          })[0];
          let download = await github.rest.actions.downloadArtifact({
              owner: context.repo.owner,
              repo: context.repo.repo,
              artifact_id: matchArtifact.id,
              archive_format: "zip",
          });
          let fs = require("fs");
          fs.writeFileSync("${{ github.workspace }}/clang-tidy-result.zip", Buffer.from(download.data));
    - name: Set environment variables
      run: |
        mkdir clang-tidy-result
        unzip clang-tidy-result.zip -d clang-tidy-result
        echo "PR_ID=$(cat clang-tidy-result/pr-id.txt)" >> "$GITHUB_ENV"
        echo "PR_HEAD_REPO=$(cat clang-tidy-result/pr-head-repo.txt)" >> "$GITHUB_ENV"
        echo "PR_HEAD_SHA=$(cat clang-tidy-result/pr-head-sha.txt)" >> "$GITHUB_ENV"
    - uses: actions/checkout@v4
      with:
        repository: ${{ env.PR_HEAD_REPO }}
        ref: ${{ env.PR_HEAD_SHA }}
        persist-credentials: false
    - name: Redownload analysis results
      uses: actions/github-script@v6
      with:
        script: |
          let artifacts = await github.rest.actions.listWorkflowRunArtifacts({
              owner: context.repo.owner,
              repo: context.repo.repo,
              run_id: ${{ github.event.workflow_run.id }},
          });
          let matchArtifact = artifacts.data.artifacts.filter((artifact) => {
              return artifact.name == "clang-tidy-result"
          })[0];
          let download = await github.rest.actions.downloadArtifact({
              owner: context.repo.owner,
              repo: context.repo.repo,
              artifact_id: matchArtifact.id,
              archive_format: "zip",
          });
          let fs = require("fs");
          fs.writeFileSync("${{ github.workspace }}/clang-tidy-result.zip", Buffer.from(download.data));
    - name: Extract analysis results
      run: |
        mkdir clang-tidy-result
        unzip clang-tidy-result.zip -d clang-tidy-result
    - name: Run clang-tidy-pr-comments action
      uses: platisd/clang-tidy-pr-comments@v1
      with:
        github_token: ${{ secrets.GITHUB_TOKEN }}
        clang_tidy_fixes: clang-tidy-result/fixes.yml
        pull_request_id: ${{ env.PR_ID }}
```

## Who's using this action?

Are you using this action in your project? Got some interesting use case?<br>
Add your project to the list below by opening a pull request or asking for it on an issue.

| Project                                                                                                        | Workflow       |
|----------------------------------------------------------------------------------------------------------------|----------------|
| [Cura](https://github.com/Ultimaker/Cura/blob/main/.github/workflows/printer-linter-pr-post.yml)               | printer-linter |
| [Orbit](https://github.com/google/orbit/blob/main/.github/workflows/report-build-and-test.yml)                 | CMake + Qt     |
| [Smartcar shield](https://github.com/platisd/smartcar_shield/blob/master/.github/workflows/tests.yml)          | CMake          |
| [cpp-command-parser](https://github.com/platisd/cpp-command-parser/blob/main/.github/workflows/clang-tidy.yml) | CMake          |
| [fheroes2](https://github.com/ihhub/fheroes2/blob/master/.github/workflows/clang_tidy_comments.yml)            | CMake          |

[clang-tidy-8 support]: https://img.shields.io/badge/clang--tidy-8-green
[clang-tidy-9 support]: https://img.shields.io/badge/clang--tidy-9-green
[clang-tidy-10 support]: https://img.shields.io/badge/clang--tidy-10-green
[clang-tidy-11 support]: https://img.shields.io/badge/clang--tidy-11-green
[clang-tidy-12 support]: https://img.shields.io/badge/clang--tidy-12-green
[clang-tidy-13 support]: https://img.shields.io/badge/clang--tidy-13-green
[clang-tidy-14 support]: https://img.shields.io/badge/clang--tidy-14-green
[clang-tidy-15 support]: https://img.shields.io/badge/clang--tidy-15-green
