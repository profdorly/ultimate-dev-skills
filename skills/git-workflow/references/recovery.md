# Undoing and recovering without losing work

Key fact: almost anything that was committed stays recoverable through the reflog for weeks, unless someone expires it (`git reflog expire`, `git gc --prune=now`). Uncommitted changes that get discarded (`reset --hard`, `checkout -- <path>`, `restore`, `clean`) are gone for good. So protect uncommitted work first, and prefer commands that refuse to destroy it.

## Protect first

Before any risky operation, make a backup and tell the user what it is.

**Git-normalized snapshot plus exact index backup.** The snapshot captures tracked and non-ignored untracked working content after Git filters; the retained index copy preserves the user's distinct staged layer. Neither changes the real index or working files.

```bash
snap_dir=$(mktemp -d "${TMPDIR:-/tmp}/git-snapshot.XXXXXX") || exit 1
printf 'snapshot temp: %s\n' "$snap_dir"       # record this literal path
real_index=$(git rev-parse --git-path index) || exit 1
work_index="$snap_dir/work.index"
index_backup="$snap_dir/original.index"
if [ -f "$real_index" ]; then
  cp "$real_index" "$index_backup" || exit 1
  cp "$real_index" "$work_index" || exit 1
fi
GIT_INDEX_FILE="$work_index" git add -A
rc=$?
if [ "$rc" -ne 0 ]; then echo "STOP: snapshot add failed; backup kept at $snap_dir"; exit "$rc"; fi
snapshot=$(GIT_INDEX_FILE="$work_index" git write-tree)
rc=$?
rm -f "$work_index"
if [ "$rc" -ne 0 ] || [ -z "$snapshot" ]; then echo "STOP: snapshot write failed; backup kept at $snap_dir"; exit 1; fi
printf 'SNAP=%s\nINDEX_BACKUP=%s\n' "$snapshot" "$index_backup"
```

Do not let cleanup hide a failed `git add` or `write-tree`. Keep `INDEX_BACKUP` until the split or risky operation is complete; it preserves staged/unstaged distinctions that a tree snapshot or filesystem copy cannot. Restore it only after inspection and approval, because replacing the live index changes staging state.

- Bring a file back from SNAP with `git restore --source=<tree> --worktree -- <path>`. Compare snapshots with `git --no-pager diff --stat --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short <tree> <other-tree>`.
- SNAP stores normalized blobs, not exact bytes: filters and line-ending conversion can make byte-different files produce the same tree. It excludes ignored files and dirty/untracked submodule contents.
- When exact bytes, ignored files, or submodule work matter, make a raw filesystem backup in a private collision-safe directory outside the repo (for example `mktemp -d` plus `cp -R -p`), record its path, and keep it until deletion is confirmed.
- The unreferenced SNAP objects may be pruned by `git gc` after about two weeks. They remain local and are not pushed. The recipe works before the first commit and in linked worktrees.

**Backup branch** (committed state):

```bash
b=$(git branch --show-current); sha=$(git rev-parse HEAD); short=$(git rev-parse --short=12 HEAD)
backup="backup/${b:-detached}-$short"
if git show-ref --verify --quiet "refs/heads/$backup"; then
  [ "$(git rev-parse "$backup")" = "$sha" ] || { echo "STOP: backup name collision"; exit 1; }
else
  git branch --no-track "$backup" "$sha" || exit 1
fi
printf 'backup: %s at %s\n' "$backup" "$sha"
```

**Stash** (tracked and untracked state, when you need a clean tree): `git stash push --include-untracked -m "backup: <reason>"`. Restore the index as well as the files with `git stash pop --index`. Ignored files remain in place. Stashing needs approval; never stash changes that aren't yours without asking.

## Safe equivalents

| You want to... | Use | Instead of | Notes |
|---|---|---|---|
| Unstage a file, keep the edits | `git restore --staged -- <path>` (before the first commit: `git reset -q -- <path>`) | `git reset --hard` | Doesn't touch working-tree content |
| Undo the last local commit, keep changes staged | `git reset --soft HEAD~1`; if it is the first commit, `git update-ref -d HEAD` | `git reset --hard HEAD~1` | Only if not pushed; approval needed |
| Undo the last local commit, keep changes unstaged | `git reset HEAD~1`; for the first commit, delete `HEAD` as above, then `git reset -q` | `git reset --hard HEAD~1` | Only if not pushed; approval needed |
| Get the tree back to the last commit | `git stash push --include-untracked -m "..."` | `git reset --hard HEAD` | Restore with `git stash pop --index`; approval needed |
| Move a branch pointer while protecting tracked/untracked edits | `git reset --keep <target>` after checking ignored-path collisions | `git reset --hard <target>` | `--keep` refuses tracked/untracked overwrites but can replace ignored files; back up collisions first |
| Discard edits in one file | A snapshot first, then `git restore -- <path>` | `git checkout -- <path>` | Destroys that file's edits; approval needed |
| Remove untracked files | `git clean -n` first, then delete only listed files with approval | `git clean -fdx` | `-x` also deletes ignored files such as `.env` |
| Remove a worktree | Inspect with `git -C <path> status --short --ignored -uall`, then remove with approval | Blind `git worktree remove` | Plain remove silently deletes ignored files; `--force` also deletes modified/untracked files |
| Undo a pushed commit | `git revert` (message format in [conventional-commits.md](conventional-commits.md#reverts)) | `reset` + force-push | New commit; history stays intact |
| Fix the last commit (not pushed) | `git commit --amend --no-edit`, or `--amend -F <file>` for a new message | Amending pushed commits | If an approved amend hook fails, prove HEAD unchanged and retry the amend; never amend after a failed ordinary commit |
| Fix an older commit (not pushed) | A `fixup!` commit plus the scripted autosquash below | Manual interactive rebase | History rewrite: approval needed |

### Fix an older local commit

Do this only when the target and every descendant are unpushed. First review the raw staged diff (`git --no-pager diff --staged --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short`), the raw unstaged diff with the same flags, and `git status --short -uall`; status alone cannot reveal an unrelated staged hunk in the same file. The staged diff must contain exactly the correction and everything else must be clean. Inspect relevant ignored files too: autosquash checks out old trees and can overwrite an ignored path that those trees tracked. Back up any ignored file that matters.

Record the target's parent as `BASE`; use the root variant when the target is the first commit. Before committing, run `git rev-list --min-parents=2 <BASE>..HEAD` (or over the full history for `--root`). If it prints any merge commit, stop: ordinary interactive rebase flattens merge topology, and this recipe intentionally does not attempt a merge-preserving rewrite.

```bash
before=$(git rev-parse HEAD) || exit 1
if ! git commit --fixup=<sha>; then echo "STOP: fixup commit failed"; exit 1; fi
if [ "$(git rev-parse HEAD^)" != "$before" ] || ! git log -1 --format=%s | grep -q '^fixup! '; then
  echo "STOP: HEAD is not the expected fixup commit"; exit 1
fi
if [ -n "$(git status --short -uall)" ]; then
  git reset --soft "$before"              # approved rollback; keeps the correction staged
  echo "STOP: handle the remaining work before rebasing"; exit 1
fi
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash <BASE>
# target is the first commit instead:
GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash --root
```

`GIT_SEQUENCE_EDITOR=true` makes `-i` non-interactive and works on git 2.31+. Without `-i`, autosquash is effective only on git 2.44 and later. If the fixup commit itself fails (for example, a hook rejects it), stop: the correction remains staged and no rollback is needed. Only reset to the recorded `before` OID after proving that HEAD advanced by exactly the expected fixup. Include that rollback in the approval proposal.

If the rebase conflicts, follow [conflicts.md](conflicts.md). Afterwards, inspect the rewritten target, confirm the correction is there, and run `git --no-pager log --format=%s <BASE>..HEAD` (or the full log for `--root`): no `fixup!` subject may remain. Run the project checks again.

## Commands that can lose data

Each one needs the user's confirmation after you show what would be lost (SKILL.md, rule 2):

| Command | What it can destroy |
|---|---|
| `git reset --hard` | Uncommitted changes in tracked files |
| `git checkout -- <path>`, `git checkout .`, `git checkout <rev> -- <path>` | Uncommitted edits in those paths |
| `git restore <path>`, `git restore .`, `git restore --source=<rev> <path>` | Uncommitted edits in those paths |
| `git checkout -f`, `git switch -f` / `--discard-changes` | All uncommitted changes in tracked files |
| `git clean -f`, `-fd`, `-fdx` | Untracked files (`-x`: ignored ones too, such as `.env`) |
| `git rm -f`, `git mv -f` | Uncommitted edits in the file, or the file being overwritten |
| `git stash drop`, `git stash clear` | Stashed work (recoverable for a while; see below) |
| `git branch -D`, `git tag -d`, `git update-ref -d` | Refs to commits that may exist nowhere else |
| `git worktree remove` | Ignored files in that worktree; `--force` also removes modified and untracked files |
| `git push --force`, `-f`, `+<refspec>`, `--mirror` | Remote commits pushed by others |
| `git push --delete <ref>`, `git push origin :<ref>` | Remote branches and tags |
| `git reflog expire`, `git gc --prune=now`, `git prune` | The reflog and unreachable objects: the safety net for everything above |
| `git filter-repo`, `git filter-branch`, BFG | Every commit gets rewritten; everyone must re-clone |

## Scenarios

**Committed on the wrong branch (not pushed):**

```bash
original=$(git branch --show-current)             # must be non-empty
git --no-pager log --oneline @{u}..HEAD           # without upstream, inspect the exact last <n> commits
git branch feat/the-work HEAD                      # new branch keeps the commits
git switch --no-overwrite-ignore feat/the-work     # same tree; local files stay here
git branch -f "$original" HEAD~<n>                # move the now-unchecked-out original branch
```

Verify both branch tips and the working tree afterwards. Moving the unchecked-out original ref avoids `reset --keep`, which can silently replace ignored files. If the misplaced commits aren't exactly the last `<n>`, or the original branch is checked out in another worktree, stop and propose another plan.

**Detached HEAD with new commits:**

```bash
git switch --no-overwrite-ignore -c rescue/<name>   # keeps them on a branch before switching away
```

**Lost a commit or deleted a branch:**

```bash
git --no-pager reflog -30                        # HEAD history
git --no-pager reflog show <branch>              # a branch's history, if the ref still exists
git branch recovered/<name> <sha>                # re-point a branch at the found commit
```

**Dropped or cleared a stash:**

```bash
git fsck --no-reflogs --unreachable | awk '/unreachable commit/ {print $3}' \
  | xargs -n1 git --no-pager log -1 --format='%h %ci %s' 2>/dev/null | grep -E ' (WIP on|On) '
git stash apply --index <sha>
```

**Undo a rebase or merge you just finished (not pushed):**

1. Create a backup branch first (see "Protect first").
2. Find the pre-operation state in `git --no-pager reflog`, for example `HEAD@{5}`, or use `ORIG_HEAD` right after the operation.
3. Before `git reset --keep <that-sha>`, compare ignored local paths with paths tracked by the target and move collisions to a unique private backup. Then run the approved reset; `--keep` protects tracked/untracked edits but not ignored files.

**Find the commit that introduced a bug.** `bisect` checks out old commits, which can overwrite ignored files such as `.env`. Run it in a throwaway worktree so the user's tree stays untouched:

```bash
bisect_tmp=$(mktemp -d "${TMPDIR:-/tmp}/git-bisect.XXXXXX") || exit 1
printf 'bisect temp: %s\n' "$bisect_tmp"   # record it; use literals if variables do not survive tool calls
bisect_wt="$bisect_tmp/worktree"
if ! git worktree add --detach "$bisect_wt" HEAD; then rmdir "$bisect_tmp"; exit 1; fi
bisect_rc=0
(
  cd "$bisect_wt" || exit 1
  <install dependencies and create safe local config>
  # Record HEAD as BAD. Confirm <command> fails here for the expected reason.
  # Switch to <known-good-sha>, confirm it passes, then switch back to BAD.
  git bisect start
  git bisect bad <BAD>
  git bisect good <known-good-sha>
  git bisect run <command>          # 0 = good, 125 = skip, other nonzero = bad
  run_rc=$?
  found=$(git rev-parse refs/bisect/bad 2>/dev/null || true)
  git bisect reset
  reset_rc=$?
  printf 'bisect candidate: %s\n' "$found"
  [ "$run_rc" -eq 0 ] && [ "$reset_rc" -eq 0 ] && [ -n "$found" ]
) || bisect_rc=$?
cleanup_rc=0
git worktree remove --force "$bisect_wt" || cleanup_rc=$?
[ "$cleanup_rc" -ne 0 ] || rmdir "$bisect_tmp" || cleanup_rc=$?
[ "$bisect_rc" -eq 0 ] || { echo "ERROR: bisect failed ($bisect_rc)"; exit "$bisect_rc"; }
[ "$cleanup_rc" -eq 0 ] || { echo "ERROR: bisect cleanup failed ($cleanup_rc)"; exit "$cleanup_rc"; }
```

Do not start until the same command passes on the known-good commit and fails for the expected reason on BAD. A missing dependency or `.env` otherwise marks commits as bad and can produce a convincing but wrong answer. If a per-commit setup precondition can fail, wrap it so that failure exits 125 rather than classifying the commit as bad. The worktree is throwaway, so its forced removal is allowed.

Other read-only helpers:

- `git --no-pager log -S'<string>' --oneline` finds commits that added or removed a string.
- `git --no-pager log --oneline -- <path>` shows a file's history.
- `git --no-pager blame -L <start>,<end> -- <path>` shows who last changed each line.

## A secret was committed

1. **Stop.** Tell the user right away which file, which commit, and whether it was pushed. Name the key; never paste its value.
2. **If it was pushed** anywhere, including a fork or CI logs, treat the secret as leaked. The user must revoke and rotate it now. Rewriting history doesn't un-leak it.
3. **If it wasn't pushed**, record the commit that contains the secret as `SECRET_COMMIT`, propose the fix, and apply it only with approval. Make sure nothing else is staged first.
   - **A whole file in the last commit** (such as `.env`): `git rm --cached -- <file>` untracks it and keeps it on disk. Add it to `.gitignore` (check that the file ends with a newline first) and stage that file. First inspect `git rev-list --parents -n1 HEAD`. For a merge commit (more than one parent), never reset to `HEAD^`: that would discard the merge topology and make other-parent commits unreachable. Preserve all parents with `git commit --amend --allow-empty --no-edit`. For a one-parent commit, compare the exact index tree (`git write-tree`) with its parent tree (`git show -s --format=%T HEAD^`); if the OIDs match, use `git reset --soft HEAD^`, otherwise amend normally. For a root commit, compare the index tree with the empty tree from `git hash-object -t tree /dev/null`; if they match, use `git update-ref -d HEAD`, otherwise amend. Tree-OID comparisons inspect raw objects and cannot be fooled by diff configuration.
   - **A value inside a tracked file in the last commit:** replace it with whatever the project uses for configuration (an environment variable or ignored config file), stage the correction, then use the same parent-count and exact-tree decision above. Never collapse a merge commit to its first parent merely because their trees match.
   - **A whole file in an older commit:** make the removal and `.gitignore` change as a fixup commit, then use "Fix an older local commit." Before rebasing, protect the now-ignored working copy in a unique directory inside the git dir; a fixed backup name can overwrite another file:

     ```bash
     secret_path='<file>'
     saved_dir=$(mktemp -d "$(git rev-parse --git-dir)/secret-backup.XXXXXX") || exit 1
     printf 'secret backup: %s\n' "$saved_dir"   # record the literal path before the rebase
     mv -- "$secret_path" "$saved_dir/value" || exit 1
     # run the scripted autosquash; after it completes or aborts:
     if [ -e "$secret_path" ] || [ -L "$secret_path" ]; then
       echo "STOP: restore destination exists; both copies preserved"; exit 1
     fi
     mkdir -p "$(dirname "$secret_path")" || exit 1
     mv -- "$saved_dir/value" "$secret_path" && rmdir "$saved_dir"
     ```

     Never overwrite a path created during the rebase or by a hook while restoring; report both copies and ask. Verify that no `fixup!` commit remains and that `git --no-pager log --all -- <file>` finds no reachable commit containing that path.
   - **A secret value inside a tracked file in an older commit:** replace the value and make a fixup commit, but do **not** move the tracked file or expect its path to disappear from history. Run the scripted autosquash, verify the corrected file at each reachable ref that matters, and run the repository's secret scanner across reachable history with redaction. Never put the secret value itself on a command line or print it while verifying.

   After any amend or rebase, run `git for-each-ref --format='%(refname)' --contains <SECRET_COMMIT>`. Any listed local branch, remote-tracking branch, or tag still reaches the old secret commit: stop, report those refs, and do not push. Rewriting or deleting those refs is a separate approved operation. The old object may also remain recoverable through the local reflog until it expires.
4. **Rewriting published history** (`git filter-repo`, BFG) is a separate, destructive step. Only run it on explicit request. Everyone with a clone has to re-clone or reset afterwards.
