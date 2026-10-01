# Conventional Commits

This is a summary of [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) in our own words, plus the type list that most tooling uses (`@commitlint/config-conventional`). If the repo has a commitlint config or a documented convention, that wins.

## Structure

```
<type>[optional scope][optional !]: <description>
<blank line>
[optional body]
<blank line>
[optional footer(s)]
```

## Rules from the spec

1. **Header:** starts with a type, optionally followed by a scope in parentheses, an optional `!`, then `: ` (colon and space), then the description.
2. **Types:** a commit that adds a feature must use `feat`, and one that fixes a bug must use `fix`. Other types are allowed.
3. **Scope:** a noun naming the part of the codebase, in parentheses, for example `fix(parser):`.
4. **Description:** a short summary that follows immediately after `: `.
5. **Body:** free-form, starts one blank line after the description, and may have several paragraphs.
6. **Footers:** start one blank line after the body.
   - Each footer is `Token: value` or `Token #value`.
   - Tokens use `-` instead of spaces (`Reviewed-by`, `Co-authored-by`), except `BREAKING CHANGE`.
7. **Breaking changes:**
   - Mark them with `!` before the colon, a `BREAKING CHANGE: <description>` footer, or both.
   - `BREAKING-CHANGE` is a valid synonym for the footer token.
   - When `!` is used, the footer can be omitted, and the description should then explain the break.
8. **Case:** types and other tokens are case-insensitive, except `BREAKING CHANGE`, which must be uppercase. Use lowercase types for consistency.

## Types (conventional-commits tooling default)

| Type | Use for | Not for |
|---|---|---|
| `feat` | New user-visible capability | Internal refactors |
| `fix` | Correcting wrong behavior | Changing intended behavior (that's `feat` or breaking) |
| `docs` | Documentation only | Code comments inside a behavior change |
| `style` | Formatting, whitespace, semicolons; no logic change | CSS or UI styling (that's `feat` or `fix`) |
| `refactor` | Restructuring with no behavior change | Performance work (`perf`) |
| `perf` | Performance improvement | Anything that changes output |
| `test` | Adding or fixing tests only | Test changes that ship with a feature (put them in the `feat` commit) |
| `build` | Build system, packaging, external dependencies | CI pipeline config |
| `ci` | CI configuration and scripts | Build tooling |
| `chore` | Maintenance that fits nothing else (tooling config, housekeeping) | Anything that fits a more specific type |
| `revert` | Reverting a previous commit | Fixing forward (`fix`) |

## Choosing a scope

- Reuse scopes already in history: `git --no-pager log --format=%s | sed -nE 's/^[a-z]+\(([^)]+)\).*/\1/p' | sort | uniq -c | sort -rn | head`.
- Omit the scope when a change spans many areas; don't invent a new scope for each commit.
- Use the lowercase kebab-case module or package name, such as `auth`, `api`, or `cli`.

## Writing the description

- Use the imperative mood ("add", "fix", "remove"), as if completing "If applied, this commit will ...".
- Start with a lowercase word and don't end with a period.
- Keep the whole header, prefix included, at 72 characters or fewer; 50 is better.
- Be specific: "fix(api): return 404 for unknown task id" rather than "fix: bug".

## Writing the body

- Explain why the change was needed and any trade-offs. The diff already shows what changed.
- Wrap lines at 72 columns. Bullet lists are fine.
- Mention side effects, migrations, or follow-ups.

## Footers you'll use

| Footer | Meaning |
|---|---|
| `BREAKING CHANGE: <what broke and how to migrate>` | Triggers a major version bump |
| `Refs: #123` or `Refs: 676104e, a215868` | References issues or commits without closing them |
| `Closes: #123` / `Fixes: #123` | GitHub and GitLab close the issue when the commit reaches the default branch |
| `Co-authored-by: Name <email>` | Credits a human co-author (GitHub recognizes it) |
| `Reviewed-by: Name <email>` | Records a reviewer |

## Reverts

The spec's FAQ recommends the `revert` type with a footer referencing the reverted commits. Git's default `Revert "..."` message doesn't follow that format, so:

```bash
git revert --no-commit <sha>
git commit -F - <<'EOF'
revert: add bulk task import

The importer creates duplicate tasks when the CSV has a BOM.
Reverting until the parser is fixed.

Refs: <sha>
EOF
```

Both commands need user approval, like any commit. To revert several commits in one go, pass them all to `git revert --no-commit` and list them all in `Refs:`. If the revert conflicts, resolve it as in [conflicts.md](conflicts.md), then run the same `git commit -F`.

## Mapping commits to Semantic Versioning

This is the one place the mapping lives; [release-versioning.md](release-versioning.md) uses it.

| Commits since the last release contain | Bump |
|---|---|
| Any breaking change (`!` or `BREAKING CHANGE`), of any type | MAJOR |
| At least one `feat`, and no breaking change | MINOR |
| Only `fix`, `perf`, or `revert`, and no `feat` or breaking change | PATCH |
| Only `docs`/`style`/`test`/`ci`/`chore`/`build`/`refactor` | No release needed, unless the project releases anyway |

- `revert` is a PATCH by default in release tools such as semantic-release. If it takes away something consumers already have, such as a released feature, judge it by what it removes: that can be breaking.
- For how `0.y.z` versions treat breaking changes, see [release-versioning.md](release-versioning.md#1-semantic-versioning-in-brief).

## Quick self-check

Default types only. Extend the list if the repo defines more. The check also enforces a lowercase start, no trailing period, and a header of 72 characters or fewer:

```bash
HEADER='fix(auth): reject expired refresh tokens'    # the first line of your message
if printf '%s\n' "$HEADER" | grep -Eq '^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\([a-z0-9._/-]+\))?!?: [a-z0-9`].*[^.]$' \
   && [ ${#HEADER} -le 72 ]; then echo "header ok"; else echo "header breaks the convention"; fi
```

Before committing, confirm:

- [ ] The type matches the change (see the table above).
- [ ] The scope is reused from history, or omitted.
- [ ] The description is imperative, lowercase, and has no period; the header is 72 characters or fewer.
- [ ] The body explains why, wrapped at 72 columns.
- [ ] A breaking change is marked with `!` and/or a `BREAKING CHANGE:` footer that includes migration notes.
- [ ] Issue references use footers.

## Examples

Good:

```
feat(import): accept CSV files with a UTF-8 BOM

Exports from Excel start with a byte-order mark, which made the header
row fail validation.

Closes: #88
```

```
refactor(auth)!: drop support for API-key headers

BREAKING CHANGE: requests must send `Authorization: Bearer <token>`.
The `X-Api-Key` header is ignored. See docs/migrate-auth.md.
```

Bad, and why:

| Message | Problem |
|---|---|
| `update stuff` | No type, and says nothing |
| `feat: Added new endpoint.` | Past tense, capitalized, trailing period |
| `fix: fix tests and add export and bump deps` | Three changes in one commit; split it |
| `chore: add OAuth login` | Wrong type; a new capability is `feat` |
