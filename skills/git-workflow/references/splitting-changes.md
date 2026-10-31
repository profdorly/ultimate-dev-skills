# Splitting a messy working tree into atomic commits

Goal: turn a tree with mixed changes into a sequence of clean, atomic commits without interactive staging and without losing anything. Work from the repository root (SKILL.md, "Orient first").

Shell variables usually don't survive between an agent's tool calls, so the steps print IDs for you to note and reuse literally. Temporary files go in `${TMPDIR:-/tmp}`.

## 1. Snapshot first (no history writes)

Take a Git-normalized snapshot plus the exact index backup from [recovery.md](recovery.md#protect-first). Record both SNAP and INDEX_BACKUP; the latter preserves the user's distinct staged layer before step 4 unstages anything. Keep it until the whole split is verified. SNAP excludes ignored files and dirty/untracked submodule contents, and filters or line-ending conversion mean equal tree IDs prove equal normalized Git content, not byte-identical files. If those distinctions matter, make the raw filesystem backup described in `recovery.md` too.

- Note the current commit as BASE: `git rev-parse --verify --quiet HEAD || echo "no commits yet"`.
- The normalized snapshot does not protect ignored files. Don't run `git clean`, remove a worktree, or stash with `--all` without showing them and asking.
- If something was already staged before you started, the user may have staged it on purpose (SKILL.md, rule 9). Ask before unstaging it.

## 2. Inventory and group by intent

```bash
git status --short -uall
git --no-pager diff --stat --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short
git --no-pager diff --staged --stat --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short
git --no-pager diff --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short -- <file>  # never print secrets
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

**Part of a file.** Without `add -p`, build a patch that contains only the hunks you want and apply it to the index. The flags make the patch independent of the user's diff configuration: color, external diff tools, relative paths, textconv, custom prefixes, and whitespace handling can otherwise corrupt it or make it silently skip paths.

```bash
split_tmp=$(mktemp -d "${TMPDIR:-/tmp}/git-split.XXXXXX") || exit 1
printf 'split temp: %s\n' "$split_tmp"   # record this literal path; variables may not survive the next tool call
git --no-pager diff --no-color --no-ext-diff --no-relative --no-textconv --ignore-submodules=none --submodule=short \
  --src-prefix=a/ --dst-prefix=b/ -- internal/tasks/service.go > "$split_tmp/service-full.patch"
cp "$split_tmp/service-full.patch" "$split_tmp/service-part.patch"
# Cut service-part.patch down (rules below).
git apply --cached --recount --whitespace=nowarn --check "$split_tmp/service-part.patch"   # dry run
git apply --cached --recount --whitespace=nowarn "$split_tmp/service-part.patch"
git --no-pager diff --staged --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short -- internal/tasks/service.go
if git diff --staged --quiet --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short -- internal/tasks/service.go; then
  echo "ERROR: nothing was staged"; exit 1
else
  rc=$?; [ "$rc" -eq 1 ] || { echo "ERROR: staged diff check failed"; exit "$rc"; }
fi
```

Run these commands from the repository root. `git apply --cached` can exit successfully while silently ignoring patch paths outside the current subdirectory; an empty staged diff is therefore a failure.

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
verify_tmp=$(mktemp -d "${TMPDIR:-/tmp}/git-verify.XXXXXX") || exit 1
printf 'verify temp: %s\n' "$verify_tmp"   # record it if checks run in later tool calls
verify_wt="$verify_tmp/worktree"
if ! git worktree add --detach "$verify_wt" HEAD; then rmdir "$verify_tmp"; exit 1; fi
check_rc=0
( cd "$verify_wt" && <install deps if needed> && <project checks> ) || check_rc=$?
cleanup_rc=0
git worktree remove --force "$verify_wt" || cleanup_rc=$?
[ "$cleanup_rc" -ne 0 ] || rmdir "$verify_tmp" || cleanup_rc=$?
[ "$check_rc" -eq 0 ] || { echo "ERROR: isolated checks failed ($check_rc)"; exit "$check_rc"; }
[ "$cleanup_rc" -eq 0 ] || { echo "ERROR: verification cleanup failed ($cleanup_rc)"; exit "$cleanup_rc"; }
```

You created this worktree only for checks, so forcing its removal, including generated or ignored check output, is allowed here. Never apply that exception to a user's worktree.

If a commit fails in isolation, it depends on something from a later group. Report it and propose a fix: reorder or merge groups. Do not start the fixup/rebase recipe while later groups are still in the working tree; rebase refuses a dirty tree and would leave a stray `fixup!` commit. Finish the approved groups first, or take a snapshot and stash the remaining work with approval, then follow "Fix an older local commit" in [recovery.md](recovery.md).

## 6. Prove nothing was lost

Take a second Git-normalized snapshot with the same recipe and compare its tree ID with SNAP:

- Same ID: tracked and non-ignored untracked content is equivalent after Git's clean filters and line-ending normalization.
- Different: `git --no-pager diff --stat --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short <SNAP> <new-tree>` shows normalized changes. Stop and investigate. If `.gitignore` changed, newly ignored files appear deleted from the snapshot even when still on disk.
- If you made the raw filesystem backup recommended in step 1, compare it as well before deleting it; that is the check for exact bytes, ignored files, and nested submodule work.

Then run `git status --untracked-files=all`: whatever you left out of the plan should still be there, unstaged, and no operation should be in progress. Report the unique temporary paths you created; remove only paths from this invocation after their purpose is complete.

## If something goes wrong

- A wrong hunk got staged: `git restore --staged -- <file>` (before the first commit: `git reset -q -- <file>`), then redo that file.
- A commit is wrong and not pushed: with approval, use `git reset --soft HEAD~1`; if it is the first commit, use `git update-ref -d HEAD`. Both keep the changes staged for restaging.
- A file in the tree looks wrong: stop and tell the user. `git restore --source=<SNAP> --worktree -- <path>` puts back its snapshot version.
