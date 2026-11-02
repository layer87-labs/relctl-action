# AGENTS.md

Instructions for AI coding agents working in this repository (GitHub Action and reusable workflows for [relctl](https://github.com/layer87-labs/relctl)). `CLAUDE.md` points here.

## Public repository — hard rules

This repository is public. Everything committed, and everything written to issues,
pull requests, comments, commit messages and release notes, is world-readable.

- Never include internal information from private projects of the owning organisation:
  no private repository names or links, no internal hostnames, URLs or addresses, no
  customer or tenant names, no ticket or issue references into private systems, no
  secrets, tokens or credentials, no internal process names or standards.
- Issues may be *motivated* by internal needs, but must be written generically:
  describe the problem and the proposal, not where it came from. Use neutral
  examples (`example-org/example-repo`).
- If in doubt, leave it out. If something internal slipped in, remove it from the
  text and tell a maintainer — edit history on GitHub may still show it.
- Write all text in English (code, comments, docs, commits, issues, PRs).

## Notes

- Reusable workflows cannot exceed the permissions of their caller; document every permission a workflow needs in `README.md`.
- Pin third-party actions and the relctl setup action to immutable references where the existing files do.
- Behaviour changes in the CLI are made in the `relctl` repository; keep inputs/outputs here in sync with it.
- Develop on a feature branch; open pull requests only when asked.
