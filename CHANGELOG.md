# Changelog

Notable changes to this pack. Each skill also carries its own version in `metadata.version` in its `SKILL.md`. Versions follow [Semantic Versioning](https://semver.org/).

## [0.1.0] - 2026-10-31

First public release.

### Added

- git-workflow: safe local git, with approval before commits and pushes, atomic Conventional Commits, conflict resolution, recovery without data loss, and releases.
- pull-requests: GitHub pull requests and CI with the `gh` CLI, including failed-check triage, review threads, and guarded merges.
- database-migrations: lock-safe, backward-compatible schema and data changes, with target-database checks and commands for 17 migration tools.
- dependencies: package vetting before install and supply-chain settings, across 14 package managers.
- shell-scripts: safe, portable shell for Linux and macOS.
- debugging: reproduce-first, root-cause debugging with regression tests.
- testing: running and writing tests without weakening them, flaky tests, snapshots, and coverage.
- code-review: evidence-based review and self-review.
- secure-coding: safe patterns at trust boundaries.
- A Claude Code plugin marketplace (`.claude-plugin/`).

[0.1.0]: https://github.com/profdorly/ultimate-dev-skills/releases/tag/v0.1.0
