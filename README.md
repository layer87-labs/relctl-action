# relctl-action

[![GitHub release](https://img.shields.io/github/v/release/layer87-labs/relctl-action)](https://github.com/layer87-labs/relctl-action/releases)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

GitHub Action and reusable workflows for [`relctl`](https://github.com/layer87-labs/relctl) — the successor to awesome-ci.

Provides:
- **Composite action** — installs the `relctl` binary (no Node.js or Docker required)
- **Reusable workflow** `generate-build-infos` — collects PR metadata (version, SHA, branch, …)
- **Reusable workflow** `create-release` — creates a GitHub release (or a release candidate) from a merged PR
- **Reusable workflow** `release-gate` — checks a release pull request (reviewed-release flow)
- **Reusable workflow** `release-forward` — brings the production branch back into the development branch

---

## Composite Action

### Setup only

```yaml
- uses: layer87-labs/relctl-action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Setup + run a command in one step

```yaml
- uses: layer87-labs/relctl-action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    run: "relctl pr info"
```

The `run` input is optional. When set, relctl is installed and the command is executed immediately. All `RELCTL_*` variables set by the command are written to `GITHUB_ENV` and `GITHUB_OUTPUT` automatically.

### Inputs

| Input            | Required | Default    | Description                                                               |
| ---------------- | -------- | ---------- | ------------------------------------------------------------------------- |
| `github-token`   | **yes**  | —          | GitHub token used for release API calls (`${{ secrets.GITHUB_TOKEN }}`)   |
| `relctl-version` | no       | `"latest"` | relctl version to install: `"latest"`, `"1.2.3"` or `"v1.2.3"` (matched against release tags with or without a `v` prefix). |
| `run`            | no       | `""`       | Optional relctl command to execute after installation.                    |

### What the action does

1. **Resolves the version** — if `"latest"`, queries the GitHub Releases API to find the most recent tag; an explicit version is looked up as a release tag with and without a `v` prefix, and the action fails with the tags it tried if neither exists.
2. **Detects the runner platform** — supports `linux/amd64`, `linux/arm64`, `darwin/amd64`, `darwin/arm64`.
3. **Downloads the binary** — from `https://github.com/layer87-labs/relctl/releases/download/<version>/`.
4. **Verifies the SHA-256 checksum** — against the `sha256sum.txt` file published with every release. Fails fast if the checksum does not match.
5. **Adds the binary to `PATH`** — available as `relctl` for all subsequent steps.
6. **Smoke-tests** the installation by running `relctl --version`.
7. **Runs the optional command** (if `run` is set).

---

## Reusable Workflows

Callers of the reusable workflows get the action version that the workflow pins, which is the previous release (a commit cannot contain its own hash).

### `generate-build-infos` — Collect PR metadata

Drop-in replacement for `fullstack-devops/git-workflows/.github/workflows/generate-build-infos.yml`.

```yaml
jobs:
  build-info:
    uses: layer87-labs/relctl-action/.github/workflows/generate-build-infos.yml@v1
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

The calling workflow must grant `contents: read` and `pull-requests: write`; relctl needs the latter to post its help comment on the pull request. A reusable workflow cannot exceed the permissions of its caller, so a caller granting only `pull-requests: read` makes the run fail at startup. Pull requests from forks get a read-only token and cannot receive the comment.

#### Inputs

| Input            | Required | Default                                   | Description                              |
| ---------------- | -------- | ----------------------------------------- | ---------------------------------------- |
| `prnumber`       | no       | `${{ github.event.pull_request.number }}` | PR number (auto-detected from event)     |
| `relctl-version` | no       | `"latest"`                                | relctl version to install                |

#### Outputs

| Output           | Description                                        |
| ---------------- | -------------------------------------------------- |
| `pr`             | Pull request number                                |
| `sha`            | SHA of the PR head commit                          |
| `sha-short`      | Short SHA of the PR head commit                    |
| `branch`         | PR source branch name                              |
| `owner`          | Repository owner                                   |
| `repo`           | Repository name                                    |
| `patch-level`    | Detected semver patch level (`major/minor/patch`)  |
| `version`        | Build version string (`<next-version>-pr-<number>`) |
| `latest-version` | Latest released version                            |
| `next-version`   | Next release version derived from PR labels        |

---

### `create-release` — Create a GitHub release

Drop-in replacement for `fullstack-devops/git-workflows/.github/workflows/create-release.yml`.

```yaml
jobs:
  release:
    uses: layer87-labs/relctl-action/.github/workflows/create-release.yml@v1
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

#### Inputs

| Input            | Required | Default    | Description               |
| ---------------- | -------- | ---------- | ------------------------- |
| `relctl-version` | no       | `"latest"` | relctl version to install |
| `version-scheme` | no       | `"semver"` | Version scheme to use: `"semver"`, `"calver"`, `"conventional-commits"` or `"changelog"` (needs relctl 2.6.0 or newer) |
| `runner`         | no       | `"ubuntu-latest"` | GitHub Actions runner label |
| `prerelease`     | no       | `false`    | Create a release candidate `X.Y.Z-rc.N` (GitHub prerelease) with `relctl release create --prerelease`; needs relctl 2.7.0 or newer |

The calling workflow must grant `contents: write` (and `pull-requests: read`, which relctl uses to resolve the merged pull request). A reusable workflow cannot exceed the permissions of its caller.

```yaml
# release candidate on every push to dev, release on main
jobs:
  release:
    permissions:
      contents: write
      pull-requests: read
    uses: layer87-labs/relctl-action/.github/workflows/create-release.yml@v1
    with:
      version-scheme: changelog
      prerelease: ${{ github.ref_name == 'dev' }}
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

#### Outputs

| Output           | Description                              |
| ---------------- | ---------------------------------------- |
| `release-id`     | ID of the created GitHub release; empty when nothing was created |
| `sha`            | SHA of the merge commit                  |
| `sha-short`      | Short SHA of the merge commit            |
| `owner`          | Repository owner                         |
| `repo`           | Repository name                          |
| `version`        | Released version (the candidate version with `prerelease`); empty when nothing was created |
| `latest-version` | Latest released version (before release) |
| `next-version`   | The version that was released            |

#### Gating later jobs

`release-id` is empty when nothing was released: relctl 2.10.2 and newer export no release id for a release that is already published, or for a candidate of a version that is already released. Build and publish jobs must therefore be guarded:

```yaml
jobs:
  release:
    uses: layer87-labs/relctl-action/.github/workflows/create-release.yml@v1
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}

  build:
    needs: [release]
    if: needs.release.outputs.release-id != ''
    runs-on: ubuntu-latest
    steps:
      - run: echo "building ${{ needs.release.outputs.version }}"
```

---

### `release-gate` — Check a release pull request

For the reviewed-release flow: a required check for the pull request that releases (for example `dev` to `main`). It runs `relctl release gate`, which verifies that the topmost `CHANGELOG.md` heading is dated and has notes, that its version is not released yet and (by default) that a release candidate points to exactly the checked commit.

```yaml
on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  gate:
    if: github.head_ref == 'dev'
    uses: layer87-labs/relctl-action/.github/workflows/release-gate.yml@v1
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

Required caller permissions: `contents: read`.

| Input               | Required | Default                | Description |
| ------------------- | -------- | ---------------------- | ----------- |
| `relctl-version`    | no       | `"latest"`             | relctl version to install |
| `runner`            | no       | `"ubuntu-latest"`      | GitHub Actions runner label |
| `require-candidate` | no       | `true`                 | Require a release candidate tag on the checked commit (`--require-candidate=false` when `false`) |
| `sha`               | no       | pull request head SHA  | Commit to check out and gate |

---

### `release-forward` — Bring the production branch back into the development branch

Runs `relctl release forward` after a release: if the development branch already contains the production branch nothing happens, if it is an ancestor it is fast-forwarded, otherwise a pull request is opened. Merge that pull request with a merge commit, not a squash.

```yaml
on:
  push:
    branches: [main]

jobs:
  forward:
    permissions:
      contents: write
      pull-requests: write
    uses: layer87-labs/relctl-action/.github/workflows/release-forward.yml@v1
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

Required caller permissions: `contents: write` and `pull-requests: write`. A fast-forward push to a protected branch needs a token that may push there; otherwise a pull request is opened.

| Input            | Required | Default           | Description |
| ---------------- | -------- | ----------------- | ----------- |
| `relctl-version` | no       | `"latest"`        | relctl version to install |
| `runner`         | no       | `"ubuntu-latest"` | GitHub Actions runner label |
| `from`           | no       | from `.relctl.yaml` | Branch to bring forward (`--from`, only passed when set) |
| `to`             | no       | from `.relctl.yaml` | Branch that receives it (`--to`, only passed when set) |

| Output         | Description |
| -------------- | ----------- |
| `result`       | `nothing`, `fast-forward` or `pull-request` |
| `pull-request` | Number of the opened pull request; empty unless `result` is `pull-request` |

---

## Pinning relctl

`relctl-version: "latest"` resolves to the newest relctl release on every run. For reproducible pipelines pin an exact version in every workflow and action call:

```yaml
    with:
      relctl-version: "2.10.2"
```

---

## Supported platforms

| OS    | Architecture |
| ----- | ------------ |
| Linux | amd64        |
| Linux | arm64        |
| macOS | amd64        |
| macOS | arm64        |

Windows runners are **not** supported (shell-only composite action).

---

## License

Apache-2.0 — see [LICENSE](LICENSE).