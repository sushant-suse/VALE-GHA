# Vale Linting GitHub Action

## Purpose

This GitHub Action automatically **lints documentation files** in pull requests using [Vale](https://errata-ai.github.io/vale/), a syntax-aware, customizable style checker.

It ensures that our documentation adheres to the [**SUSE Style Guide**](https://documentation.suse.com/style/current/html/style-guide-adoc/index.html), catching style issues, spelling inconsistencies, and formatting errors **before merging**.

## Motto

*“Write consistently, review effortlessly.”*  

The goal is to help developers and technical writers maintain high-quality, consistent documentation without manual style reviews.

## How It Works

1. Triggered on every **pull request** that modifies documentation files (`.adoc`, `.md`, `.txt`) in the `docs/` folder (currently targeting files under `docs/` folder).
2. Checks out the repository and the official **SUSE Vale Style Guide**.
3. Installs dependencies: **Vale, jq, Asciidoctor, curl, tar**.
4. Configures Vale with a custom `.vale.ini` pointing to the style guide.
5. Lints only the **changed documentation files** in the PR.
6. Posts a **detailed comment** on the PR showing:
   - Number of errors and warnings
   - File-by-file breakdown
   - Line numbers, messages, and suggestions
   - Link to the style guide for each rule

If there are no issues, the action posts a friendly confirmation that everything is following the style guide.

## GitHub Action Workflow

### Workflow Header

```yaml
name: Vale Linting
````

* Names this GitHub Action workflow **“Vale Linting”**.
* Displayed in the GitHub Actions tab.

```yaml
on:
  pull_request:
    paths:
      - 'docs/**/*.adoc'
      - 'docs/**/*.md'
      - 'docs/**/*.txt'
```

* The workflow triggers **only on pull requests**.
* It checks **only the documentation files** in the `docs/` folder with extensions `.adoc`, `.md`, and `.txt`.

```yaml
permissions:
  contents: read
  pull-requests: write
  issues: write
```

* Defines the **permissions** this workflow needs:

  * Read repository contents
  * Post or update pull request comments
  * Create or update issues (if needed)

### Job Definition

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
```

* Defines a job named `lint` that runs on a **latest Ubuntu runner**.

### Step 1: Checkout the repository

```yaml
- name: Checkout Docs Repository
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
```

* Checks out the PR branch repository to the workflow runner.
* `fetch-depth: 0` ensures **full Git history**, which is needed to compare changes.

### Step 2: Checkout SUSE Vale Style Guide

```yaml
- name: Checkout SUSE Style Guide
  uses: actions/checkout@v4
  with:
    repository: 'openSUSE/suse-vale-styleguide'
    path: 'suse-styles'
```

* Checks out the official **SUSE Vale Style Guide** into the `suse-styles` folder.
* Vale will use these rules to lint documentation.

### Step 3: Install Dependencies

```yaml
- name: Install Vale, jq, and Asciidoctor
  run: |
    sudo apt-get update
    sudo apt-get install -y asciidoctor jq curl tar

    VALE_VERSION="3.12.0"
    VALE_URL="https://github.com/errata-ai/vale/releases/download/v${VALE_VERSION}/vale_${VALE_VERSION}_Linux_64-bit.tar.gz"
    mkdir -p /tmp/vale-install
    curl -L "$VALE_URL" | tar -xz -C /tmp/vale-install/
    sudo mv /tmp/vale-install/vale /usr/local/bin/
    sudo chmod +x /usr/local/bin/vale
```

* Installs required tools:
  * `asciidoctor` – converts `.adoc` files if needed
  * `jq` – parses JSON
  * `curl` & `tar` – download and extract Vale
* Downloads and installs **Vale** version `3.12.0`.

### Step 4: Configure Vale

```yaml
- name: Configure Vale
  run: |
    rm -f suse-styles/common/Spelling.yml
    echo "StylesPath = $GITHUB_WORKSPACE/suse-styles" > .vale.ini
    echo "MinAlertLevel = warning" >> .vale.ini
    echo "[*.adoc]" >> .vale.ini
    echo "BasedOnStyles = common, asciidoc" >> .vale.ini
    echo "[*.md]" >> .vale.ini
    echo "BasedOnStyles = common" >> .vale.ini
    echo "[*.txt]" >> .vale.ini
    echo "BasedOnStyles = common" >> .vale.ini
```

* Removes the default `Spelling.yml` to **avoid conflicts**.
* Creates `.vale.ini` pointing to the `suse-styles` folder.
* Defines which **styles to apply per file type**.

### Step 5: Run Vale on changed files & Post PR Comment

```yaml
- name: Run Vale and Post/Update PR Comment
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    set -e
    cd "$GITHUB_WORKSPACE"
```

* Runs the linting process in the repository root.
* Uses `GITHUB_TOKEN` to authenticate **commenting on the PR**.

#### Fetch changed files

```bash
CHANGED_FILES=$(curl -s -H "Authorization: token $GITHUB_TOKEN" \
  "https://api.github.com/repos/${GITHUB_REPOSITORY}/pulls/$PR_NUMBER/files" \
  | jq -r '.[] | select(.filename | test("^docs/.*\\.(adoc|md|txt)$")) | .filename')
```

* Queries GitHub API to get **only changed documentation files** in the PR.
* Skips linting if **no docs files changed**.

#### Run Vale

```bash
VALE_OUTPUT=$(echo "$CHANGED_FILES" | xargs /usr/local/bin/vale --config="$GITHUB_WORKSPACE/.vale.ini" --output=JSON --no-exit || true)
```

* Runs Vale **only on changed files**.
* Outputs **JSON** (easier to parse in shell scripts).
* `--no-exit` ensures the workflow continues even if there are errors.

#### Post PR comment

* Fetches existing comments by GitHub Actions to **update instead of spamming**.
* If **no issues**, posts a ✅ success message.
* If **issues exist**, generates a Markdown table with:
  * File, Line, Severity, Message, Suggestion, Rule, Rule Link
* Uses `<details>` to allow reviewers to **expand/collapse the full list of issues**.
* Maximum issues displayed are controlled by `MAX_ISSUES` (currently 50).

## Configuration Options

* **MAX_ISSUES** – Change the number of issues shown in the PR comment or remove the limit entirely.
* **File types** – Update the `paths` in the workflow trigger to include other file types if needed.
* **Style guide** – Point to a custom style guide by changing the repository in Step 2.
