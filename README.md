# Create Verified Commit and Tag

Create verified, signed commits (and optional annotated tags) using the GitHub API — no GPG keys required.

## Why choose this action?

Commits created by `git push` in workflows appear as **unverified** in the GitHub UI. This action uses the GitHub GraphQL `createCommitOnBranch` mutation, which GitHub signs automatically — every commit shows the **Verified** badge, attributed to your `github-actions[bot]` or GitHub App identity.

### Advantages over similar actions

**Works on any runner OS.** This is a JavaScript action, not Docker. Docker-based actions are Linux-only. This action runs identically on Ubuntu, macOS, and Windows runners with no extra setup.

**No need to hardcode file paths.** The `files` input accepts glob patterns matched against `git status` output. Write `dist/**` or `src/**/*.ts` once and the action resolves what actually changed at runtime — no fragile file lists to maintain as your build tooling evolves.

**Commit and tag in one step.** Create a verified commit and an annotated tag together, both attributed to the same identity. Pairing tags with [GitHub immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases) gives your releases cryptographic attestations without requiring GPG or SSH keys.

**Idempotent workflows.** `fail-on-empty: false` combined with the `committed` boolean output lets you skip downstream steps when nothing changed — without failing the run or creating empty commits. Re-running a release workflow is always safe.

## Usage

### Minimal — commit all changes on a push event

```yaml
- uses: oWretch/create-signed-commit@v1.0.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    commit-message: 'chore: update generated files'
```

### Specific files using glob patterns

```yaml
- uses: oWretch/create-signed-commit@v1.0.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    commit-message: 'chore: bump version to ${{ steps.version.outputs.new_version }}'
    files: |
      package.json
      package-lock.json
      CHANGELOG.md
```

> **Note:** `files` globs are matched against the files reported by `git status`, not a filesystem scan. Only files already changed in the workspace will be committed.

### With an annotated tag (releases)

```yaml
- uses: oWretch/create-signed-commit@v1.0.0
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    commit-message: 'chore: release v1.2.3'
    tag-name: v1.2.3
    tag-message: 'Release v1.2.3'
```

### Commit-if-changed pattern

```yaml
- uses: oWretch/create-signed-commit@v1.0.0
  id: commit
  with:
    token: ${{ secrets.GITHUB_TOKEN }}
    commit-message: 'chore: update generated files'
    fail-on-empty: false
- if: steps.commit.outputs.committed == 'true'
  run: echo "Committed ${{ steps.commit.outputs.commit-sha }}"
```

### Pull request workflow

```yaml
on:
  pull_request:

jobs:
  format:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - run: npm run format
      - uses: oWretch/create-signed-commit@v1.0.0
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          commit-message: 'style: auto-format'
```

> **Note:** Use `ref: github.event.pull_request.head.sha` in the checkout step to avoid checking out the synthetic merge ref. Fork PRs are not supported with `GITHUB_TOKEN`.

## Integration examples

### Signed release commits with Node.js semantic-release

Replace `@semantic-release/git` with a local plugin that calls this action so
that every release commit is **Verified** by a GitHub App instead of a plain
git push.

**`.releaserc.json`**

```json
{
  "plugins": [
    "@semantic-release/commit-analyzer",
    "@semantic-release/release-notes-generator",
    ["@semantic-release/npm", { "npmPublish": false }],
    "./.github/semantic-release-sign-plugin.cjs",
    "@semantic-release/github"
  ]
}
```

**`.github/semantic-release-sign-plugin.cjs`**

```js
'use strict'
const { execFileSync } = require('child_process')

async function prepare(_config, context) {
  const { logger, cwd: releaseCwd, nextRelease } = context
  const cwd = releaseCwd || process.cwd()
  const token = process.env.RELEASE_APP_TOKEN
  if (!token) throw new Error('RELEASE_APP_TOKEN is required')

  logger.log('Rebuilding dist...')
  execFileSync('npm', ['run', 'build'], { cwd, stdio: 'inherit' })

  const message = `chore(release): ${nextRelease.version} [skip ci]\n\n${nextRelease.notes}`
  logger.log('Creating signed release commit...')
  execFileSync('node', ['dist/index.js'], {
    cwd,
    stdio: 'inherit',
    env: {
      ...process.env,
      GITHUB_WORKSPACE: cwd,
      INPUT_TOKEN: token,
      'INPUT_COMMIT-MESSAGE': message,
      INPUT_FILES: 'package.json\ndist/index.js',
      INPUT_REF: 'refs/heads/main',
      'INPUT_FAIL-ON-EMPTY': 'false',
    },
  })

  logger.log('Syncing local HEAD to signed commit...')
  execFileSync('git', ['fetch', 'origin', 'main'], { cwd, stdio: 'inherit' })
  execFileSync('git', ['reset', '--hard', 'origin/main'], { cwd, stdio: 'inherit' })
}

module.exports = { prepare }
```

The plugin runs in the `prepare` lifecycle — after `@semantic-release/npm` has
bumped `package.json` but before `@semantic-release/github` creates the tag and
GitHub Release. The `git fetch + reset` at the end ensures the tag is placed on
the signed commit rather than the original trigger commit.

The `RELEASE_APP_TOKEN` environment variable must be the GitHub App installation
token (not `GITHUB_TOKEN`), so that commits are attributed to your App.

---

### Signed release commits with python-semantic-release

Use `--no-commit --no-push --no-tag` to let `python-semantic-release` prepare
files without creating a git commit, then use this action to create the Verified
commit and annotated tag.

```yaml
- name: Bump version (no commit)
  id: psr
  run: |
    uv run semantic-release version --no-commit --no-push --no-tag --no-changelog
    echo "version=$(uv run semantic-release version --print)" >> $GITHUB_OUTPUT

- name: Build distribution
  run: uv run --with build python -m build

- name: Create signed release commit
  id: commit
  uses: oWretch/create-signed-commit@v1.0.0
  with:
    token: ${{ secrets.APP_TOKEN }}
    commit-message: 'chore(release): ${{ steps.psr.outputs.version }} [skip ci]'
    files: |
      pyproject.toml
      uv.lock

- name: Create signed release tag
  uses: oWretch/create-signed-commit@v1.0.0
  with:
    token: ${{ secrets.APP_TOKEN }}
    commit-message: 'Tagging ${{ steps.psr.outputs.version }}'
    tag-name: v${{ steps.psr.outputs.version }}
    tag-message: 'Release v${{ steps.psr.outputs.version }}'
    commit-sha: ${{ steps.commit.outputs.commit-sha }}
    fail-on-empty: false

- name: Publish to PyPI
  uses: pypa/gh-action-pypi-publish@release/v1
```

> **Note:** `fail-on-empty: false` on the tag step is needed because no new
> files are being committed — the tag is created against an existing commit SHA.

---

## Permissions

```yaml
permissions:
  contents: write
```

If committing changes to workflow files under `.github/workflows/`, also add:

```yaml
permissions:
  contents: write
  workflows: write
```

## Inputs

| Name             | Required | Default        | Description                                                                                     |
| ---------------- | -------- | -------------- | ----------------------------------------------------------------------------------------------- |
| `token`          | ✅       | —              | GitHub token with `contents: write` permission                                                  |
| `commit-message` | ✅       | —              | Commit message. First line is the headline; subsequent lines become the body.                   |
| `ref`            | ❌       | Auto-detect    | Target branch name. Auto-detected from the workflow event context.                              |
| `files`          | ❌       | All changes    | Multiline glob patterns to filter which changed files to include in the commit                  |
| `tag-name`       | ❌       | —              | Annotated tag name to create (e.g. `v1.0.0`)                                                    |
| `tag-message`    | ❌       | Commit message | Annotated tag message                                                                           |
| `fail-on-empty`  | ❌       | `true`         | Fail the step if no changed files are detected. Set to `false` for commit-if-changed workflows. |

## Outputs

| Name         | Description                                                    |
| ------------ | -------------------------------------------------------------- |
| `commit-sha` | SHA of the created commit (empty if no changes were committed) |
| `tag-sha`    | SHA of the created tag object (empty if no tag was created)    |
| `committed`  | `true` if a commit was created, `false` if skipped             |

## How commits are verified

This action calls the GitHub GraphQL `createCommitOnBranch` mutation. GitHub signs every commit created through this API on behalf of the authenticated identity, so commits automatically display the **Verified** badge in the GitHub UI — no GPG or SSH signing keys are needed.

- With `GITHUB_TOKEN`: commits are signed and attributed to `github-actions[bot]`
- With a GitHub App installation token: commits are signed and attributed to `your-app[bot]`

Tags are created using the GitHub REST Git Data API and are **not** cryptographically signed. Pair annotated tags with [GitHub immutable releases](https://docs.github.com/en/code-security/concepts/supply-chain-security/immutable-releases) for supply chain attestations.

## Limitations

- **Tags are unsigned:** Annotated tags are created via the REST API and carry no cryptographic signature.
- **Fork PRs:** `GITHUB_TOKEN` has read-only access to fork repositories. Fork PRs are not supported; use a GitHub App token with explicit write access if needed.
- **Symlinks and submodules:** Not supported.
- **chmod-only changes:** File mode changes are not detected by `git status` in this context and will not be committed.
- **LFS-tracked files:** Files managed by Git LFS are not supported.
- **Payload size:** All changed files are held in memory. The total commit payload is subject to a ~150 MB limit.

## Versioning and updates

This action uses [immutable releases](https://docs.github.com/en/repositories/releasing-projects-on-github/managing-releases-in-a-repository) — there are no floating `@v1` or `@v2` tags. Each release is a pinned version (e.g. `@v1.0.0`) that never moves.

**Always pin to a specific version:**

```yaml
- uses: oWretch/create-signed-commit@v1.0.0
```

**Enable Dependabot to receive automatic update PRs:**

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
```

With Dependabot configured, version bump PRs are raised automatically whenever a new release is published. All Dependabot PRs are signed commits that pass the CI checks.

## License

[Apache 2.0](LICENSE)
