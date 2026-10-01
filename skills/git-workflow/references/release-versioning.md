# Releases, versions, tags, and changelogs

Commits are how developers track change. A version is how consumers track it. As soon as anything depends on the code (a published package, another team, a deployed client), give each release a version number, a tag, and a changelog entry.

## 0. Follow the project's release tooling

Check for release automation before doing anything by hand:

- `semantic-release` (`.releaserc*`, `release` key in `package.json`)
- `release-please` (`release-please-config.json`, `.release-please-manifest.json`)
- `changesets` (`.changeset/`)
- `commit-and-tag-version` / `standard-version`
- `cargo-release`, `bump-my-version` / `bumpversion`, `goreleaser`
- a `release` target in the Makefile or a release workflow in CI

If one exists, use it and don't tag or edit versions manually. Every release step below that writes history (commit, tag, push) needs user approval.

## 1. Semantic Versioning in brief

[SemVer 2.0.0](https://semver.org/) uses `MAJOR.MINOR.PATCH`:

| Part | Bump when |
|---|---|
| MAJOR | Consumers must change their code or config to upgrade (breaking) |
| MINOR | New backward-compatible functionality |
| PATCH | Backward-compatible bug fixes |

Other details:

- **Pre-releases:** `1.4.0-rc.1`, `2.0.0-beta.2`. They sort before the final release.
- **Build metadata:** `1.4.0+build.57`. It's ignored for precedence.
- **`0.y.z` (initial development):** anything may change. Many projects and release tools bump MINOR for breaking changes there; that's a tooling convention, not part of SemVer. Follow the project's history and tooling.

**Breaking means observable, not large.** A one-line change to behavior that consumers rely on (output format, error type, default value, ordering, timing) is breaking, even if the diff is tiny (Hyrum's law). When unsure, treat it as breaking and ask the user.

## 2. Find the last release and the tag format

```bash
git --no-pager tag --list --sort=-v:refname | head -10   # the tag format in use
git describe --tags --abbrev=0 --match 'v[0-9]*'          # nearest release tag behind HEAD; adjust the pattern to the format
```

Match what you see. If tags look like `v1.2.3`, use the `v` prefix; if they're bare `1.2.3`, don't. In monorepos, follow package-scoped tags such as `pkg-name@1.2.3`. If `git describe` finds nothing, this is the first release.

## 3. Decide the bump

```bash
git --no-pager log <last-tag>..HEAD --no-merges --format='%h %s%n%b'   # first release: drop "<last-tag>.."
```

Apply the table in [conventional-commits.md](conventional-commits.md#mapping-commits-to-semantic-versioning). Also read the diffs of vague or non-conventional commits: the message may understate what changed.

## 4. Update the version in one place

- **Version derived from tags** (setuptools-scm, hatch-vcs, `git describe` at build time): only tag.
- **Version stored in a manifest** (`package.json`, `pyproject.toml`, `Cargo.toml`, `*.csproj`): update that single source in a release commit, `chore(release): 1.4.0`, together with the changelog. Use the ecosystem's command (`npm version 1.4.0 --no-git-tag-version`, `poetry version 1.4.0`, `cargo set-version 1.4.0` from cargo-edit) so lockfiles and derived files stay in sync.
- Don't hand-edit the version in several scattered files. If the project does that, report it as a risk.

## 5. Tag

Tags need approval. Use annotated tags, which carry an author, a date, and a message:

```bash
git tag -a v1.4.0 -m "Release 1.4.0"
git push origin <branch>          # the release commit reaches the remote branch first
git push origin v1.4.0            # then only this tag, not --tags
```

`git push --atomic origin <branch> v1.4.0` does both in one step: if the host rejects either ref, neither is updated.

- Never move, delete, or re-create a tag that has been pushed. Consumers and caches may already depend on it. If a release is broken, ship the next patch version.
- Tag the commit that actually contains the version bump and the changelog.

## 6. Changelog

Use the [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) layout unless the project has its own:

```markdown
# Changelog

## [Unreleased]

## [1.4.0] - 2026-09-28
### Added
- Bulk task import from CSV files.
### Changed
- Task lists are sorted by due date by default.
### Deprecated
- `GET /v1/tasks/all`; use the paginated `GET /v1/tasks`. Removal planned for 2.0.
### Fixed
- Recurring tasks no longer drift by an hour after daylight-saving changes.
```

- The groups are Added, Changed, Deprecated, Removed, Fixed, and Security. Include only the ones with entries.
- Newest version first, with ISO dates (`YYYY-MM-DD`).
- Write for consumers: what changed for them and whether they need to act. Don't paste the commit log.
- Add entries to `[Unreleased]` in the same change that makes them, while the impact is fresh. At release time, rename `[Unreleased]` to the new version and date.
- Breaking changes get a migration note: what to change, from what, to what. Deprecations name a replacement and a planned removal version.
- Optionally, end the file with comparison links such as `[1.4.0]: <repo-url>/compare/v1.3.0...v1.4.0`.

## Release checklist

- [ ] Release tooling checked (step 0); manual steps only if none exists.
- [ ] Bump matches what consumers can observe: breaking is MAJOR, additive is MINOR, fix is PATCH.
- [ ] Version updated in its single source; lockfiles consistent.
- [ ] Changelog entry curated, grouped, dated, with migration notes for breaking changes.
- [ ] Project checks pass on the release commit.
- [ ] Annotated tag in the project's format, on the release commit.
- [ ] Release commit pushed to its branch, then only that tag; both with approval.
- [ ] Reported: version, tag, commit hash, and what was pushed.
