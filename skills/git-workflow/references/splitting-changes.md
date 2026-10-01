# Splitting a messy working tree into atomic commits

Goal: turn a tree with mixed changes into a sequence of clean, atomic commits without interactive staging and without losing anything.

Shell variables usually don't survive between an agent's tool calls, so the steps print IDs for you to note and reuse literally. Temporary files go in `${TMPDIR:-/tmp}`.

## 1. Snapshot first (no history writes)

Take a snapshot of the working tree, the same recipe as in [recovery.md](recovery.md#protect-first). It includes untracked files and touches neither the index nor your files. Note the tree ID it prints as SNAP:

```bash
idx="${TMPDIR:-/tmp}/git-snapshot-$$.index"
cp "$(git rev-parse --git-path index)" "$idx" 2>/dev/null || rm -f "$idx"
GIT_INDEX_FILE="$idx" git add -A && GIT_INDEX_FILE="$idx" git write-tree
rm -f "$idx"
```

- Note the current commit as BASE: `git rev-parse --verify --quiet HEAD || echo "no commits yet"`.
- Ignored files aren't in the snapshot. Staging never deletes them, but don't run `git clean` or `stash -u` on them without asking.
- If something was already staged before you started, the user may have staged it on purpose (SKILL.md, rule 9). Ask before unstaging it.

## 2. Inventory and group by intent

```bash
git status --short -uall
git --no-pager diff --stat
git --no-pager diff --staged --stat
git --no-pager diff -- <file>      # one file at a time; never print .env*, keys, or credential files
```

Group hunks by intent, not by file. One file can hold hunks for two different commits. Typical groups:

- refactors and renames that later changes depend on;
- the behavior change, with its tests;
- formatting-only changes;
- dependency or lockfile updates;
- docs.

Order the commits so each one builds and passes checks on its own: prerequisites first, dependents after.

Leave these out of the plan and point them out: changes that existed before you started, and anything that looks like a secret or junk (SKILL.md, rule 7).

## 3. Propose the plan, then wait

```
Commit plan (base 1a2b3c4, branch refactor/task-service):
1. refactor(tasks): extract validation into a Validator type
   internal/tasks/service.go (hunks 1-2), internal/tasks/validator.go (new)
2. feat(tasks): reject tasks with due dates in the past
   internal/tasks/validator.go (hunk 3), internal/tasks/validator_test.go
3. style(tasks): apply gofmt
   internal/tasks/list.go
Left uncommitted: notes.md (not mine), .env (secret; add it to .gitignore?)
```

Execute only after approval.

## 4. Stage each commit

Start from an empty index, unless the user asked you to keep what they staged:

```bash
git reset -q
```

Without a commit argument, `git reset` only unstages. It doesn't move the branch or touch any file, and unlike `git restore --staged .`, it also works before the first commit.

**Whole files:**

```bash
git add -- internal/tasks/validator.go internal/tasks/validator_test.go
```

**Part of a file.** Without `add -p`, build a patch that contains only the hunks you want and apply it to the index. The extra flags keep user config out of the way: colors, external diff tools, and `diff.noprefix` would break the patch, and `apply.whitespace` could reject it or silently change what gets staged.

```bash
git --no-pager diff --no-color --no-ext-diff --src-prefix=a/ --dst-prefix=b/ -- internal/tasks/service.go > "${TMPDIR:-/tmp}/service-full.patch"
# Copy it to service-part.patch and cut the copy down (rules below).
git apply --cached --recount --whitespace=nowarn --check "${TMPDIR:-/tmp}/service-part.patch"   # dry run
git apply --cached --recount --whitespace=nowarn "${TMPDIR:-/tmp}/service-part.patch"
git --no-pager diff --staged -- internal/tasks/service.go                                      # verify
```

Cutting the patch down:

- Keep the header lines (`diff --git`, `index`, `---`, `+++`).
- Delete the `@@` hunks you don't want, whole.
- Inside a hunk you keep, leave an added line out by deleting it, and leave a removed line out by turning its leading `-` into a space. Don't touch context lines.
- `--recount` fixes the `@@` line counts. A bad cut fails `--check` and changes nothing.
- `--cached` changes only the index; the working file keeps all its changes.

**Part of a new (untracked) file:** run `git add -N -- <file>` (intent-to-add) first. The file then shows up in `git diff`, and the same technique works.

## 5. Commit and verify each one

Commit with the approved message (`git commit -F -` with a heredoc; see [conventional-commits.md](conventional-commits.md)).

The working tree still holds all the other changes, so running tests there checks everything, not the commit alone. To check a single commit in isolation, use a throwaway worktree:

```bash
git worktree add --detach "${TMPDIR:-/tmp}/verify-<short-sha>" HEAD
( cd "${TMPDIR:-/tmp}/verify-<short-sha>" && <install deps if needed> && <project checks> )
git worktree remove "${TMPDIR:-/tmp}/verify-<short-sha>"
```

If the checks left untracked files there, `remove` refuses. You created this worktree and it holds nothing but check output, so `git worktree remove --force` is fine here, and only here.

If a commit fails in isolation, it depends on something from a later group. Report it and propose a fix before continuing: reorder or merge groups, or fix the unpushed commit (with approval; see "Fix an older commit" in [recovery.md](recovery.md)).

## 6. Prove nothing was lost

Take a second snapshot with the same recipe and compare its tree ID with SNAP:

- Same ID: the working tree is exactly as it was, untracked files included.
- Different: `git --no-pager diff --stat <SNAP> <new-tree>` shows what changed. Stop and investigate before doing anything else. If you changed `.gitignore` in between, files it now ignores show up as deleted; check that they're still on disk.

Then check `git status`: whatever you left out of the plan should still be there, unstaged.

## If something goes wrong

- A wrong hunk got staged: `git restore --staged -- <file>` (before the first commit: `git reset -q -- <file>`), then redo that file.
- A commit is wrong and not pushed: with approval, `git reset --soft HEAD~1`, then restage. The changes stay in the tree.
- A file in the tree looks wrong: stop and tell the user. `git restore --source=<SNAP> --worktree -- <path>` puts back its snapshot version.
