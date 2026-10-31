---
name: git-workflow
description: Safe, disciplined local git for coding agents. Atomic commits with Conventional Commits messages, short-lived branches, worktrees for parallel work, conflict resolution, splitting a messy working tree into clean commits, undoing and recovering work without data loss, and semver releases with tags and changelogs. Use when about to stage, commit (including "commit everything"), branch, stash, merge, rebase, cherry-pick, pull, push, resolve conflicts, discard or recover changes, bisect, manage worktrees, tag a release, or write a changelog. Commits, history rewrites, and pushes always require explicit user approval. Not for GitHub pull requests, CI, or the gh CLI.
license: MIT
compatibility: Requires git 2.31 or later and a POSIX shell. A few recipes note newer versions.
metadata:
  author: profdorly
  version: "0.1.0"
  derived-from: "addyosmani/agent-skills git-workflow-and-versioning (MIT)"
---

# Git Workflow

Git is the user's safety net. Keep history clean and reviewable, and never destroy work, including work you did not create.

## Safety rules (non-negotiable)

1. **Ask before recording, rewriting, or moving work.** Never commit, amend, merge, pull, rebase, cherry-pick, revert, stash, tag, push, delete a branch, remove a worktree, or run a `reset` that moves a branch without the user's explicit approval in this conversation. Propose first (see "Commit proposal"), then wait. Approving a plan covers exactly the actions in that plan. "Implement X" is not approval to commit X; for requests like "commit everything", see "Commits". Read-only commands (`status`, `diff`, `log`, `show`), `git fetch`, and staging your own changes need no approval; creating a branch or worktree belongs in the plan you propose.
2. **Never discard work without confirmation.** Anything that overwrites or deletes uncommitted changes, untracked or ignored files, stashes, branches, tags, the reflog, or remote refs can lose data: `reset --hard`, `checkout -- <path>`, `checkout .`, `restore <path>` (without `--staged`), `checkout -f`, `switch --discard-changes`, `clean -f`, `stash drop`/`clear`, `branch -D`, any `worktree remove` when that worktree has ignored files, `worktree remove --force` (except on a throwaway worktree you created for checks), `push --delete`, `reflog expire`, `gc --prune=now`, and the others listed in [references/recovery.md](references/recovery.md). Before any of them, show what would be lost and offer a reversible alternative from that file.
3. **Never force-push with `--force`, `-f`, or a `+<refspec>`.** Only the lease forms in "Pushing" are allowed, and only with approval. On the default branch or a shared branch, only when the user explicitly asks, after the backup-and-confirm steps there.
4. **Never bypass hooks** unless the user asks: no `--no-verify`, no `git commit -n`, no `-c core.hooksPath=...`, no `HUSKY=0`, `SKIP=...`, or `LEFTHOOK=0`, and no editing or deleting hook files. When a hook fails, follow "When a hook fails".
5. **Never change git config** (identity, editor, credentials, signing, hooks, `core.*`, `rerere`) unless asked, and don't override those settings for a single command with `-c`, environment variables, `--no-gpg-sign`, or `--author`. Exceptions: rule 6's prompt-suppressing environment and `GIT_SEQUENCE_EDITOR=true` for the non-interactive autosquash recipe; the upstream that `git push -u` records is also allowed.
6. **Never run interactive commands.** They hang agents. Avoid `add -i`/`-p`/`-e`, `checkout -p`, `restore -p`, `reset -p`, `stash -p`, `clean -i`, `mergetool`, `difftool`, `git help` or `--help` (use `-h`), `commit` or `tag -a` without `-m`/`-F`, and interactive rebase. The one allowed `rebase -i` is the scripted autosquash in [references/recovery.md](references/recovery.md), with `GIT_SEQUENCE_EDITOR=true` so no editor opens. Instead use:
   - `git --no-pager` for anything that prints history, diffs, or lists;
   - `git commit -F <file>` for messages, and `--no-edit` on `merge`, `pull`, `revert`, and `commit --amend` when the default message is right;
   - `GIT_EDITOR=true` in front of any `--continue` (`git rebase --continue` opens an editor even without a terminal);
   - `GIT_TERMINAL_PROMPT=0 GCM_INTERACTIVE=0` in front of every network command (`fetch`, `pull`, `push`, `ls-remote`), so Git and Git Credential Manager fail instead of prompting. For an SSH remote, also set `GIT_SSH_COMMAND='ssh -o BatchMode=yes'`; if `core.sshCommand` is configured, append `-o BatchMode=yes` to that command instead of replacing it.
7. **Never commit secrets or junk.** Watch for `.env*`, keys (`*.pem`, `*.key`, `id_*`), credential files, tokens, build output, dependency folders, and large binaries the project does not track. Never force-add an ignored file (`git add -f`). Check file names before printing contents, and never repeat a secret's value: name the file and the key. If a secret shows up, stop and tell the user. If it was already committed, see [references/recovery.md](references/recovery.md).
8. **Stage deliberately.** Prefer `git add -- <paths>` after reviewing `git status`. An untracked directory shows up as one entry and can hide a `.env`; list it with `git status --short -uall -- <dir>` before adding it. Use `git add -A`, `git add .`, or `git commit -a` only when the status shows exactly what you intend to commit.
9. **Leave other people's changes alone.** Changes that existed before you started, staged or not, belong to the user. Don't stage, unstage, stash, or discard them without asking; point them out.

Repo conventions (message style, language, scopes, branch names, merge policy) override this skill's defaults, but never these rules.

## Orient first (read-only, no approval needed)

Run Git workflows from the repository root. Resolve it with `git rev-parse --show-toplevel`, then use that path as the working directory for every recipe; some pathspec and patch commands can silently do nothing from a subdirectory.

Run this before your first git write, and ideally before your first edit:

```bash
git --version
git status --untracked-files=all          # branch, upstream, operation, changed and untracked files
git --no-pager diff --stat --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short
git --no-pager diff --staged --stat --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short
git --no-pager log --oneline -10          # fails harmlessly in a repo with no commits yet
git --no-pager stash list
```

Check for:

- **Branch:** on the default branch? `git symbolic-ref --quiet --short refs/remotes/origin/HEAD` prints something like `origin/main`; the default branch is the part after `origin/`. If it prints nothing, try `git ls-remote --symref origin HEAD` with rule 6's non-interactive network environment, then look for `main` or `master`, or ask. Propose a feature branch instead of committing there. A new repository's first commits are the exception; say so in the proposal.
- **Operation in progress:** plain `git status` names a merge, rebase, cherry-pick, revert, or bisect; `git status --short` does not. If you didn't start it, stop and ask.
- **Detached HEAD:** don't commit until you know why.
- **Pre-existing changes:** anything modified before your work started belongs to the user (rule 9). If you get here after editing, changes in files you never touched are the user's, and a file you edited that also has changes you don't recognize needs a question.
- **Repo conventions:** read `git --no-pager log --format=%s -20`, `CONTRIBUTING.md`, and any commitlint or `.gitmessage` config.

## Commits

- **Atomic:** one logical change per commit. It passes the project's checks on its own and can be reverted alone.
- **Separate concerns:** refactor vs behavior change, formatting vs logic, dependency bumps vs features. Generated files go with the change that generated them.
- **Size:** about 100 changed lines is ideal, 300 is fine, and over 1000 should be split. Mechanical changes (renames, lockfiles, generated code) are the exception.
- **Mixed or messy working tree:** follow [references/splitting-changes.md](references/splitting-changes.md).

### Before proposing a commit

1. **Run the project's own checks.** Look in `Makefile`, `package.json` scripts, `pyproject.toml`/`tox.ini`, `Cargo.toml`, `go.mod`, `.pre-commit-config.yaml`, and CI workflow files. Use check-only modes (no `--fix`, `--write`, or snapshot updates), then run `git status --untracked-files=all` again: if a check changed files, review those changes before staging them. If a check can't run, say so in the proposal.
2. **Stage and review the exact diff:** use `git --no-pager diff --staged --stat --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short` first, then the same raw flags without `--stat` for files that are safe to print. This prevents external diff, textconv, or submodule diff configuration from hiding or transforming staged objects.
3. **Scan the staged diff for secrets.** This prints file names only and catches common token formats, not every secret:

   ```bash
   git --no-pager diff --staged --name-only --text --no-ext-diff --no-textconv --ignore-submodules=none --submodule=short -G'-----BEGIN [A-Z ]*PRIVATE KEY|AKIA[0-9A-Z]{16}|gh[pousr]_[A-Za-z0-9]{36}|github_pat_[A-Za-z0-9_]{22,}|xox[abposr]-[A-Za-z0-9-]{10,}|(^|[^A-Za-z0-9_-])sk-[A-Za-z0-9_-]{32,}'
   ```

   If the repo configures a secret scanner, or `gitleaks` is installed, run it as well (`gitleaks git --pre-commit --staged --redact`). Any hit: stop (rule 7).

### "Commit everything" and similar requests

A direct request such as "commit everything" or "commit this" approves one commit of your own changes without a separate proposal, provided that:

- every change in the tree is yours;
- the checks, your review, and the secret scan found nothing to leave out;
- the changes form one logical commit;
- no "Stop and ask" condition applies.

Otherwise, say what's in the way and propose a plan instead (often a split into several commits). Either way, write a proper message and report the commit afterwards.

### Message format

Use Conventional Commits unless the repo uses something else. Full rules are in [references/conventional-commits.md](references/conventional-commits.md).

```
<type>[(scope)][!]: <description>

[body: why, not what; wrap at 72 columns]

[footers: BREAKING CHANGE: ..., Refs: #123, Co-authored-by: ...]
```

- **Types:** `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`.
- **Description:** imperative mood, lowercase first word, no trailing period; the whole header 72 characters or fewer (aim for 50).
- **Body:** explain intent and trade-offs; the diff already shows what changed.
- **Breaking change:** add `!` before the colon and/or a `BREAKING CHANGE:` footer.
- **Language:** match the repo's history.
- **No AI attribution:** don't add `Co-authored-by` trailers for AI tools or "Generated with" lines unless the user or the repo asks for them.
- **How to write it:** multi-line messages go through a heredoc or a file, never an editor:

```bash
git commit -F - <<'EOF'
fix(auth): reject expired refresh tokens

The expiry check compared seconds against milliseconds, so expired
tokens were still accepted.

Refs: #142
EOF
```

### Commit proposal

Present this, then wait for approval:

```
Proposed commit 1 of 2 (branch: fix/token-expiry, new)
Files:   src/auth/refresh.py, tests/auth/test_refresh.py
Checks:  pytest passed (84 tests), ruff check passed, secret scan clean
Message:
  fix(auth): reject expired refresh tokens

  The expiry check compared seconds against milliseconds, so expired
  tokens were still accepted.
Left out: README.md (edited before I started; not part of this change)
```

After committing, confirm with `git --no-pager log -1 --stat` and `git status --untracked-files=all`, which also exposes an unfinished operation.

### When a hook fails

A failed `pre-commit` or `commit-msg` hook means the attempted commit did not happen. First verify that `HEAD` is unchanged, then:

- **A normal commit failed:** never answer by amending; that would rewrite the previous real commit. Review any hook edits, restage only this commit's files, and retry a new commit with the approved message.
- **An approved amendment failed:** do not create a new commit. After proving `HEAD` is unchanged, review any hook edits and retry the same amendment (`--amend`) under the existing approval.
- **The hook rejected the message:** fix the message in the same operation (new commit or amendment), retry, and report the change.
- **Passing needs code changes:** make them only if they're within the task, then show the new staged diff and ask again.
- **The failure is unrelated, or the hook needs a missing tool:** stop and ask.

Never use `--no-verify`. Distinguish a failed new commit from a failed amendment before choosing the retry command.

## Branches

- Don't commit on the default branch unless the user asks ("Orient first" shows how to find it).
- **Naming:** `<type>/<short-kebab-description>`, for example `feat/csv-import` or `fix/token-expiry`, unless the repo does it differently.
- **Keep branches short-lived and in sync.** Fetching needs no approval; updating your branch does. Prefer `git fetch origin`, then, with approval, `git merge --ff-only --no-overwrite-ignore @{u}`. This gives `pull --ff-only` behavior while refusing to overwrite ignored local files. If it can't fast-forward, show `git --no-pager log --oneline --left-right HEAD...@{u}` and propose a rebase or a merge.
- **Rebase vs merge:** follow the repo's policy. Without one, rebase only local, unpushed branches, and merge anything already pushed or shared. Use `--no-overwrite-ignore` with `git switch` and `git merge`; ignored files are not protected by the usual snapshot/stash and the default switch/merge behavior can replace them. Rewriting pushed history needs explicit approval.
- **Deleting a branch** needs approval. Use `git branch -d`, which refuses unmerged branches; `-D` throws away unmerged commits, so show them first (`git --no-pager log --oneline <default>..<branch>`) and get the user's OK for that loss.

## Worktrees (parallel work)

- Check whether you're already isolated: if `git rev-parse --absolute-git-dir` and `git rev-parse --path-format=absolute --git-common-dir` print different paths, you're in a linked worktree. If your harness provides worktrees natively, use those.
- **Create:** `git worktree add -b <branch> <path> <base>`. The usual path is a sibling directory, `../<repo>-<branch>`, with any `/` in the branch name replaced by `-`. It lives outside the repo, so name it in your proposal.
- **List:** `git worktree list`.
- **Remove:** removal needs approval. First run `git -C <path> status --short --ignored -uall`: plain `git worktree remove <path>` refuses modified or untracked files but silently deletes ignored files, including `.env`. Show any `!!` entries and confirm their loss; then remove. `--force` requires confirmation, except on a throwaway worktree you created yourself for checks (see [references/splitting-changes.md](references/splitting-changes.md)).
- One agent per worktree. Two agents must never edit the same worktree.

## Pushing

- Push only with approval, rule 6's non-interactive network environment, a named remote, and one explicit destination. First push: `git push -u origin HEAD:refs/heads/<branch>`.
- Never push directly to the default branch unless asked.
- A failed `pre-push` hook means nothing was pushed; handle it like a failed commit hook.
- **Rejected push** ("non-fast-forward", "fetch first"): never answer with force. Run `git fetch origin <branch>`, show the divergence with `git --no-pager log --oneline --left-right HEAD...origin/<branch>`, and propose a rebase or a merge.
- **Force push**, only with an exact lease, one remote, and one destination:
  - Your own branch, after an approved rebase:
    1. Fetch `origin <branch>` and record `remote_tip=$(git rev-parse origin/<branch>)`.
    2. Prove that remote tip was integrated before the rewrite. If it is not an ancestor of current `HEAD`, inspect the branch reflog and require `git merge-base --is-ancestor "$remote_tip" <old-branch-tip>` to succeed for at least one pre-rewrite tip. If the reflog cannot prove this, stop and integrate the remote branch instead of pushing.
    3. Push exactly one ref: `git push --force-with-lease=refs/heads/<branch>:<remote-tip-oid> origin HEAD:refs/heads/<branch>`.

    An explicit expected OID disables the protection normally added by `--force-if-includes`, so the ancestry/reflog proof is mandatory. The exact lease still fails if the remote moves after the fetch. Never omit the remote or refspec: configured wildcard push refspecs could rewrite other branches.
  - Default or shared branch, only at the user's explicit request:
    1. Fetch `origin <branch>`, record its full OID, and show what would disappear: `git --no-pager log --oneline <branch>..origin/<branch>`.
    2. Back up the reviewed tip under an OID-derived name such as `backup/<branch>-remote-<short-oid>`. If that ref exists, verify it already equals the full reviewed OID; otherwise create it with `git branch --no-track <backup> <full-oid>` and verify creation succeeded.
    3. Get a second, explicit OK for exactly the commits shown.
    4. Push one ref with the reviewed exact lease: `git push --force-with-lease=refs/heads/<branch>:<full-oid> origin HEAD:refs/heads/<branch>`. If the remote moved, the push fails; start over at step 1.
  - The host's branch protection may still refuse. Report it; don't work around it.
- Pull requests, CI, and the `gh` CLI are out of scope for this skill.

## Other operations: read the reference first

| Situation | Read |
|---|---|
| Writing or checking a commit message, or mapping commits to version bumps | [references/conventional-commits.md](references/conventional-commits.md) |
| Mixed changes in the tree to split into several commits | [references/splitting-changes.md](references/splitting-changes.md) |
| Merge, rebase, cherry-pick, or revert conflict | [references/conflicts.md](references/conflicts.md) |
| Backing up, undoing, reverting, fixing an older commit, recovering lost work, a committed secret, finding the commit that broke something, the full list of commands that can lose data | [references/recovery.md](references/recovery.md) |
| Version bump, tag, changelog, release | [references/release-versioning.md](references/release-versioning.md) |

## Final report

When you finish, report:

- **Changes made:** each file and what changed.
- **Intentionally not touched:** related issues you left out of scope.
- **Concerns:** assumptions the user should confirm.
- **Git state:** branch, commits created (short hash and header), what was pushed, what remains uncommitted or staged, and any backup branches, snapshots, or stashes you created.

Report only what you verified. Don't claim tests passed, a commit exists, or a push happened without checking (`git --no-pager log -1`, `git status --untracked-files=all`, `git --no-pager log -1 origin/<branch>`).

## Stop and ask when

- You're on the default branch with changes to commit.
- The tree has changes you didn't make.
- A command would delete untracked or ignored files, discard changes, or rewrite pushed history.
- A hook or check fails and the fix is outside the task.
- A secret, credential, or large binary appears in a diff.
- You find a detached HEAD or an unfinished operation you didn't start.
- A push is rejected.
- Commit signing fails or waits for input, or `user.name`/`user.email` isn't set.

## Tempting shortcuts

| Shortcut | Why not |
|---|---|
| "I'll commit everything at the end." | One giant commit can't be reviewed, bisected, or reverted cleanly. Propose commits per slice. |
| "`reset --hard` and redo is faster." | It also deletes the user's uncommitted work. Take a snapshot or a backup branch first ([references/recovery.md](references/recovery.md)) and use a reversible command. |
| "The hook is flaky; `--no-verify` just this once." | Hooks are the team's gate, and `HUSKY=0` or `SKIP=` is the same bypass. Fix the cause or ask. |
| "I'll just amend." | Amending pushed commits rewrites shared history, and amending after a failed hook alters the previous commit. Make a new commit. |
| "The push was rejected; I'll force it." | Forcing deletes commits someone else pushed. Fetch, show the divergence, and propose a rebase or a merge. |
| "They asked me to implement it, so committing is implied." | Implementation is not commit approval. Propose. |
| "`git add -A` is fine." | It sweeps in secrets, junk, and other people's edits. |
| "The message doesn't matter." | History is documentation for future humans and agents. Explain why. |
| "It's a small diff, so it's a patch bump." | Version by what consumers can observe, not by diff size. See [references/release-versioning.md](references/release-versioning.md). |
