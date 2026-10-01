# Resolving merge, rebase, cherry-pick, and revert conflicts

Principle: understand why each side changed before touching a hunk. Preserve both intents where possible, don't invent new behavior, and when a hunk needs a product decision, ask.

## 0. Before starting a merge or rebase

- Start from a clean tree. `git merge --abort` may not be able to restore uncommitted changes that existed before the merge. Commit or stash first, with approval.
- The merge, rebase, or cherry-pick itself needs approval (see SKILL.md, rule 1). Once approved, resolving and continuing that same operation is covered. Report the resolutions before continuing if any hunk needed a judgment call.
- Start it without an editor: `git merge --no-edit <branch>`, and for a revert, `git revert --no-commit <sha>` (message format in [conventional-commits.md](conventional-commits.md#reverts)). A plain `git rebase <upstream>` or `git cherry-pick <sha>` needs no extra flag.

## 1. Identify the situation

```bash
git status                                         # names the operation in progress
git --no-pager diff --name-only --diff-filter=U    # conflicted files
```

Which operation is in progress:

| Exists | Operation |
|---|---|
| `$(git rev-parse --git-path MERGE_HEAD)` | merge |
| `$(git rev-parse --git-path rebase-merge)` or `rebase-apply` | rebase (`REBASE_HEAD` is the commit being replayed) |
| `$(git rev-parse --git-path CHERRY_PICK_HEAD)` | cherry-pick |
| `$(git rev-parse --git-path REVERT_HEAD)` | revert |

If you didn't start the operation, stop and ask.

## 2. Know which side is which

| Operation | `ours` / stage 2 / `HEAD` | `theirs` / stage 3 |
|---|---|---|
| `git merge other` | the branch you're on | `other` |
| `git rebase main` | **`main` plus the commits already replayed** | **your commit being replayed** |
| `git cherry-pick X` | the branch you're on | `X`, the change being picked |
| `git revert X` | the branch you're on | **`X`'s parent**: the content before `X`, which the revert brings back (stage 1 is `X` itself) |

During a rebase the labels are reversed from what people expect, and in a revert `theirs` is the old content. Double-check before using `--ours`/`--theirs`.

## 3. Understand both intents

```bash
git --no-pager log --merge --oneline -- <file>        # commits on both sides touching the file
git --no-pager show REBASE_HEAD                       # during rebase: the commit being replayed
git --no-pager show :1:<file>                         # common ancestor version
git --no-pager show :2:<file>                         # ours
git --no-pager show :3:<file>                         # theirs
```

- `git log --merge` works during a merge; since git 2.45 it also works during rebase, cherry-pick, and revert.
- Read the commit messages and any linked issue or PR on both sides. The resolution depends on why each change was made.
- To see the ancestor inside the conflict markers, regenerate them in diff3 style. This rewrites the file and discards edits already made to it, so do it before editing:

  ```bash
  git checkout --conflict=diff3 -- <file>
  ```

## 4. Resolve each hunk

- **Both changes are compatible:** keep both, ordered so the code still makes sense.
- **The changes are incompatible:** pick the side that matches the goal of the operation (for example "bring this branch up to date with main without losing the feature"), and note the trade-off in your report.
- **The resolution needs a product or API decision:** stop and ask. Don't guess.
- **Deleted on one side, modified on the other:** decide deliberately. `git rm <file>` accepts the deletion; `git add <file>` keeps the modified file.
- **Taking a whole side** (`git checkout --ours/--theirs -- <file>`): only for generated files or when you've confirmed the other side's change is obsolete. Never do it blanket-style to "make it compile".
- **Lockfiles** (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `poetry.lock`, `uv.lock`, `Cargo.lock`, `go.sum`, and similar): don't merge them by hand.
  1. Resolve the manifest first (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, and so on).
  2. Check out either side's lockfile.
  3. Run the package manager's install or lock command so it re-resolves from the merged manifest.
  4. Stage the regenerated lockfile.
- Don't enable `rerere` or change `merge.conflictStyle`; that would change git config.

## 5. Verify

```bash
git --no-pager diff --check                                        # flags leftover conflict markers and whitespace errors
git --no-pager grep -nE '^(<{7}|[|]{7}|={7}|>{7})( |$)' -- <resolved files>   # belt and braces, diff3 markers included
```

Then run the project's checks (build or typecheck, tests, lint). Fix anything the merge broke. Those fixes belong to the resolution, not to a new feature.

## 6. Stage and continue (non-interactive)

```bash
git add -- <resolved files>
```

Then continue the operation:

| Operation | Command |
|---|---|
| merge | `git commit --no-edit` (uses the prepared merge message) |
| rebase | `GIT_EDITOR=true git rebase --continue` (keeps the replayed commit's message) |
| cherry-pick | `GIT_EDITOR=true git cherry-pick --continue` |
| revert | `GIT_EDITOR=true git revert --continue` |
| revert started with `--no-commit` | `git commit -F <message-file>` (see [conventional-commits.md](conventional-commits.md#reverts)) |

Rebases can stop several times. Repeat steps 1 to 6 until `git status` shows no operation in progress.

- If a replayed commit becomes empty because its change is already upstream, git will say so. Report it; skipping it (`git rebase --skip`) is fine when you've confirmed the change is already there.

## Aborting

Aborting is safe and returns to the state before the operation:

```bash
git merge --abort
git rebase --abort
git cherry-pick --abort
git revert --abort
```

It also throws away any resolutions made so far. If you've already resolved hunks, ask before aborting.

## Report

For each conflicted file, list:

- each hunk: which side you kept, or how you combined them, and why;
- any hunk where you made a judgment call (flag it for review);
- the checks you ran and their result;
- the final state: operation finished, branch, and new commits.
