# Undoing and recovering without losing work

Key fact: almost anything that was committed stays recoverable through the reflog for weeks, unless someone expires it (`git reflog expire`, `git gc --prune=now`). Uncommitted changes that get discarded (`reset --hard`, `checkout -- <path>`, `restore`, `clean`) are gone for good. So protect uncommitted work first, and prefer commands that refuse to destroy it.

## Protect first

Before any risky operation, make a backup and tell the user what it is.

**Snapshot of the working tree.** It covers tracked and untracked files and touches neither the index nor the files. Note the tree ID it prints:

```bash
idx="${TMPDIR:-/tmp}/git-snapshot-$$.index"
cp "$(git rev-parse --git-path index)" "$idx" 2>/dev/null || rm -f "$idx"
GIT_INDEX_FILE="$idx" git add -A && GIT_INDEX_FILE="$idx" git write-tree
rm -f "$idx"
```

- Bring a file back: `git restore --source=<tree> --worktree -- <path>`. Compare two snapshots: `git --no-pager diff --stat <tree> <other-tree>`.
- Ignored files, such as `.env`, aren't included.
- It's short-term: nothing references the tree, so `git gc` may prune it after about two weeks. Its objects stay in the local repository and are never pushed.
- It works before the first commit and in linked worktrees.

**Backup branch** (committed state):

```bash
b=$(git branch --show-current); git branch "backup/${b:-detached}-$(date +%Y%m%d-%H%M%S)"
```

**Stash** (uncommitted state, when you need a clean tree): `git stash push --include-untracked -m "backup: <reason>"`. It empties the working tree and is reversible with `git stash pop`. It needs approval, and never stash changes that aren't yours without asking.

## Safe equivalents

| You want to... | Use | Instead of | Notes |
|---|---|---|---|
| Unstage a file, keep the edits | `git restore --staged -- <path>` (before the first commit: `git reset -q -- <path>`) | `git reset --hard` | Doesn't touch working-tree content |
| Undo the last local commit, keep the changes staged | `git reset --soft HEAD~1` | `git reset --hard HEAD~1` | Only if not pushed; approval needed |
| Undo the last local commit, keep the changes unstaged | `git reset HEAD~1` (mixed) | `git reset --hard HEAD~1` | Only if not pushed; approval needed |
| Get the tree back to the last commit | `git stash push --include-untracked -m "..."` | `git reset --hard HEAD` | Reversible with `git stash pop`; approval needed |
| Move a branch pointer while protecting local edits | `git reset --keep <target>` | `git reset --hard <target>` | Refuses if it would overwrite local changes; approval needed |
| Discard edits in one file | A snapshot first, then `git restore -- <path>` | `git checkout -- <path>` | Destroys that file's edits; approval needed |
| Remove untracked files | `git clean -n` first (dry run), then delete only listed files with approval | `git clean -fdx` | `-x` also deletes ignored files such as `.env` |
| Undo a pushed commit | `git revert` (message format in [conventional-commits.md](conventional-commits.md#reverts)) | `reset` + force-push | New commit; history stays intact |
| Fix the last commit (not pushed) | `git commit --amend --no-edit`, or `--amend -F <file>` for a new message | Amending pushed commits | Never after a failed hook (SKILL.md, "When a hook fails") |
| Fix an older commit (not pushed) | `git commit --fixup=<sha>`, then `git rebase --autosquash <sha>~1` (`--root` if it's the first commit) | `git rebase -i` | Git 2.44+; no editor opens. History rewrite: approval needed |

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
| `git worktree remove --force` | Modified and untracked files in that worktree |
| `git push --force`, `-f`, `+<refspec>`, `--mirror` | Remote commits pushed by others |
| `git push --delete <ref>`, `git push origin :<ref>` | Remote branches and tags |
| `git reflog expire`, `git gc --prune=now`, `git prune` | The reflog and unreachable objects: the safety net for everything above |
| `git filter-repo`, `git filter-branch`, BFG | Every commit gets rewritten; everyone must re-clone |

## Scenarios

**Committed on the wrong branch (not pushed):**

```bash
git --no-pager log --oneline @{u}..HEAD   # commits not pushed yet; without an upstream, use git --no-pager log --oneline -10
git branch feat/the-work                  # a new branch keeps them
git reset --keep HEAD~<n>                 # move the current branch back past only those <n> commits
git switch feat/the-work
```

`--keep` refuses if it would overwrite local changes. If the misplaced commits aren't the last `<n>`, stop and propose a plan.

**Detached HEAD with new commits:**

```bash
git switch -c rescue/<name>              # keeps them on a branch before switching away
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
git stash apply <sha>
```

**Undo a rebase or merge you just finished (not pushed):**

1. Create a backup branch first (see "Protect first").
2. Find the pre-operation state in `git --no-pager reflog`, for example `HEAD@{5}`, or use `ORIG_HEAD` right after the operation.
3. Run `git reset --keep <that-sha>`, with approval.

**Find the commit that introduced a bug.** `bisect` checks out old commits, which can also overwrite ignored files such as `.env`. Run it in a throwaway worktree so the user's tree stays untouched:

```bash
git worktree add --detach "${TMPDIR:-/tmp}/bisect" HEAD
# then, inside ${TMPDIR:-/tmp}/bisect:
git bisect start
git bisect bad                    # current commit is broken
git bisect good <known-good-sha>
git bisect run <command>          # exit 0 = good, 125 = can't test (skip), 1-124 or 126-127 = bad
git bisect reset                  # always finish with this
# back in the repo:
git worktree remove "${TMPDIR:-/tmp}/bisect"
```

Other read-only helpers:

- `git --no-pager log -S'<string>' --oneline` finds commits that added or removed a string.
- `git --no-pager log --oneline -- <path>` shows a file's history.
- `git --no-pager blame -L <start>,<end> -- <path>` shows who last changed each line.

## A secret was committed

1. **Stop.** Tell the user right away which file, which commit, and whether it was pushed. Name the key; never paste its value.
2. **If it was pushed** anywhere, including a fork or CI logs, treat the secret as leaked. The user must revoke and rotate it now. Rewriting history doesn't un-leak it.
3. **If it wasn't pushed**, propose the fix and apply it only with approval. Make sure nothing else is staged first.
   - **A whole file in the last commit** (such as `.env`): `git rm --cached -- <file>` untracks it and keeps it on disk. Add it to `.gitignore` (check that the file ends with a newline first), run `git add -- .gitignore`, then `git commit --amend --no-edit`.
   - **A value inside a tracked file:** replace it with whatever the project uses for configuration (an environment variable, an ignored config file), stage, and amend.
   - **An older commit:** make the same fix as a fixup commit (`git commit --fixup=<sha>`) and fold it in with `git rebase --autosquash <sha>~1` (or `--root`). The rebase checks out the old commits, which can overwrite and then delete the file on disk. Move it out of the working tree first, for example `mv -- .env "$(git rev-parse --git-path saved.env)"`, and move it back afterwards, even if the rebase stops or you abort it.
4. **Rewriting published history** (`git filter-repo`, BFG) is a separate, destructive step. Only run it on explicit request. Everyone with a clone has to re-clone or reset afterwards.
