# relctl-action

[![GitHub release](https://img.shields.io/github/v/release/layer87-labs/relctl-action)](https://github.com/layer87-labs/relctl-action/releases)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

GitHub Action and reusable workflows for [`relctl`](https://github.com/layer87-labs/relctl) — the successor to awesome-ci.

Provides:
- **Composite action** — installs the `relctl` binary (no Node.js or Docker required)
- **Reusable workflow** `generate-build-infos` — collects PR metadata (version, SHA, branch, …)
- **Reusable workflow** `create-release` — creates a GitHub release from a merged PR

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
| `relctl-version` | no       | `"latest"` | relctl version to install. Use `"latest"` or a semver tag like `"1.2.3"`. |
| `run`            | no       | `""`       | Optional relctl command to execute after installation.                    |

### What the action does

1. **Resolves the version** — if `"latest"`, queries the GitHub Releases API to find the most recent tag.
2. **Detects the runner platform** — supports `linux/amd64`, `linux/arm64`, `darwin/amd64`, `darwin/arm64`.
3. **Downloads the binary** — from `https://github.com/layer87-labs/relctl/releases/download/<version>/`.
4. **Verifies the SHA-256 checksum** — against the `sha256sum.txt` file published with every release. Fails fast if the checksum does not match.
5. **Adds the binary to `PATH`** — available as `relctl` for all subsequent steps.
6. **Smoke-tests** the installation by running `relctl --version`.
7. **Runs the optional command** (if `run` is set).

---

## Reusable Workflows

### `generate-build-infos` — Collect PR metadata

Drop-in replacement for `fullstack-devops/git-workflows/.github/workflows/generate-build-infos.yml`.

```yaml
jobs:
  build-info:
    uses: layer87-labs/relctl-action/.github/workflows/generate-build-infos.yml@v1
    secrets:
      token: ${{ secrets.GITHUB_TOKEN }}
```

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

#### Outputs

| Output           | Description                              |
| ---------------- | ---------------------------------------- |
| `release-id`     | ID of the created GitHub release         |
| `sha`            | SHA of the merge commit                  |
| `sha-short`      | Short SHA of the merge commit            |
| `owner`          | Repository owner                         |
| `repo`           | Repository name                          |
| `version`        | Released version                         |
| `latest-version` | Latest released version (before release) |
| `next-version`   | The version that was released            |

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