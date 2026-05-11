# relctl-action

[![GitHub release](https://img.shields.io/github/v/release/layer87-labs/relctl-action)](https://github.com/layer87-labs/relctl-action/releases)
[![License](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)

A GitHub Composite Action that downloads, verifies, and installs the [`relctl`](https://github.com/layer87-labs/relctl) binary into the `PATH` of your workflow runner.

> **No Node.js or Docker required** — pure shell steps only.

---

## Usage

```yaml
- uses: layer87-labs/relctl-action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

### Pin to a specific version

```yaml
- uses: layer87-labs/relctl-action@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    relctl-version: "1.2.3"
```

---

## Inputs

| Input            | Required | Default    | Description                                                               |
| ---------------- | -------- | ---------- | ------------------------------------------------------------------------- |
| `github-token`   | **yes**  | —          | GitHub token used for release API calls (`${{ secrets.GITHUB_TOKEN }}`)   |
| `relctl-version` | no       | `"latest"` | relctl version to install. Use `"latest"` or a semver tag like `"1.2.3"`. |

---

## What the action does

1. **Resolves the version** — if `"latest"`, queries the GitHub Releases API to find the most recent tag.
2. **Detects the runner platform** — supports `linux/amd64`, `linux/arm64`, `darwin/amd64`, `darwin/arm64`.
3. **Downloads the binary** — from `https://github.com/layer87-labs/relctl/releases/download/<version>/`.
4. **Verifies the SHA-256 checksum** — against the `sha256sum.txt` file published with every release. Fails fast if the checksum does not match.
5. **Adds the binary to `PATH`** — available as `relctl` for all subsequent steps.
6. **Smoke-tests** the installation by running `relctl --version`.

---

## Full example workflow

```yaml
name: Release

on:
  push:
    tags:
      - "v*.*.*"

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup relctl
        uses: layer87-labs/relctl-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          relctl-version: "latest"

      - name: Create release
        run: |
          relctl release create \
            --pr-number ${{ github.event.pull_request.number }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
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