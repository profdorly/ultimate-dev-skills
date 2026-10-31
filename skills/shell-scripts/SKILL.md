---
name: shell-scripts
description: Use when writing, reviewing, or debugging shell scripts, one-liners, Makefile recipes, Dockerfile RUN lines, CI run steps, or git hooks, or when a command behaves differently on macOS and Linux. Covers quoting, strict-mode traps, guarded deletes, cleanup traps, GNU vs BSD tools (sed -i, date, stat), bash 3.2, dash, zsh, and ShellCheck.
license: MIT
metadata:
  author: profdorly
  version: "0.1.0"
---

# Shell Scripts

Shell makes the dangerous thing short: an unset variable turns `rm -rf "$dir/"` into `rm -rf /`, an unquoted file name with a space becomes two arguments, and a failed command in the middle of a pipeline goes unnoticed. Write scripts that stop on errors, quote everything, guard every delete, and behave the same on the machines they'll run on.

## Rules (non-negotiable)

1. **Quote every expansion:** `"$var"`, `"$(cmd)"`, `"${array[@]}"`. Leave one unquoted only when you want word splitting or globbing, and say so in a comment.
2. **Guard destructive commands.** Before `rm -rf`, `mv`, `find -delete`, or an overwriting redirect, make sure the path variable is set and sane: `rm -rf -- "${build_dir:?}/"`, plus a check that it isn't `/`, `$HOME`, or outside the directory the script owns. Put `--` before file arguments. For user data, prefer moving to a backup location over deleting.
3. **Fail loudly.** Bash scripts start with `set -euo pipefail`, knowing where it doesn't protect you ("Strict mode traps"). Errors go to stderr with a clear message and a nonzero exit code.
4. **Keep secrets off command lines and out of traces.** Arguments are visible in `ps`, shell history, and CI logs, and `set -x` prints them. Read secrets from environment variables or files, and never `set -x` around them.
5. **Choose the shell on purpose.** Use `#!/usr/bin/env bash` for bash features and `#!/bin/sh` only for strictly POSIX code (on Debian and Ubuntu, `/bin/sh` is dash). Scripts that must run on stock macOS can't use bash 4+ features: `/bin/bash` there is 3.2.
6. **Use portable flags, or branch on the OS.** GNU and BSD tools differ (`sed -i`, `date -d`, `stat -c`, `grep -P`, `readlink -f`, `mktemp -t`). Check [references/portability.md](references/portability.md) before using a flag in a script that runs on both.
7. **Never wait for input.** Scripts used by automation take flags, not prompts. Only pass `--yes`, `-f`, or `--force` to a destructive tool when the user approved that exact operation.
8. **Lint what you write.** Run ShellCheck on every script you create or change: `shellcheck -x <script>`, or without installing it, `docker run --rm -v "$PWD:/mnt:ro" -w /mnt koalaman/shellcheck:stable -x <script>`. Fix the findings or explain each one you keep.
9. **Make scripts that change things safe to rerun.** Check before acting (does the directory exist, is the line already in the file), offer `--dry-run` for anything destructive or slow, and ask before running a script that deletes data, uses `sudo`, or changes system configuration. Never pipe a download straight into a shell (`curl ... | sh`) without the user's approval; download, inspect, and verify a checksum first.

## Script skeleton

```bash
#!/usr/bin/env bash
# Compress *.log files older than 7 days in a directory.
set -euo pipefail

usage() {
  cat <<'EOF'
Usage: rotate-logs.sh [--dry-run] <log-dir>

Compress *.log files older than 7 days in <log-dir>.
EOF
}

die() { printf 'rotate-logs: %s\n' "$*" >&2; exit 1; }

dry_run=0
while [ $# -gt 0 ]; do
  case $1 in
    --dry-run) dry_run=1 ;;
    -h | --help) usage; exit 0 ;;
    --) shift; break ;;
    -*) die "unknown option: $1" ;;
    *) break ;;
  esac
  shift
done
[ $# -eq 1 ] || { usage >&2; exit 2; }
log_dir=$1
[ -d "$log_dir" ] || die "not a directory: $log_dir"
command -v gzip >/dev/null 2>&1 || die "gzip is required"

run() {
  if [ "$dry_run" -eq 1 ]; then
    printf 'would run:'
    printf ' %q' "$@"
    printf '\n'
  else
    "$@"
  fi
}

find "$log_dir" -type f -name '*.log' -mtime +7 -print0 |
  while IFS= read -r -d '' file; do
    run gzip -- "$file"
  done
```

It runs on bash 3.2 and later, on Linux and macOS. More building blocks (temporary directories with cleanup, locks, retries, logging, reading files line by line, the script's own directory) are in [references/patterns.md](references/patterns.md).

## Strict mode traps

`set -euo pipefail` catches a lot, but not everything:

- **`set -e` is off inside conditions.** A function called as `if f; then`, `f || x`, `f && x`, or `! f` keeps running after a failing command inside it. Check exit codes explicitly in functions used as conditions.
- **`local` and `export` hide the exit status.** `local out=$(cmd)` succeeds even when `cmd` fails. Write `local out; out=$(cmd)`.
- **`cond && action` as the last statement of a function** makes the function return 1 when `cond` is false, and the caller exits. Use `if cond; then action; fi`.
- **Arithmetic that evaluates to 0 fails.** `((count++))` when `count` is 0 returns 1, and bash 4+ exits. Write `count=$((count + 1))`.
- **`SIGPIPE` under `pipefail`.** `producer | head -n 1` can fail with status 141 when `head` exits before the producer finishes. Capture the output first (`out=$(producer)`), or handle 141 explicitly.
- **Command substitution inside arguments.** `echo "$(false)"` doesn't stop the script. Assign to a variable first, then use it.
- **Empty arrays with `set -u` on bash before 4.4** (including stock macOS): `"${args[@]}"` fails with "unbound variable". Write `${args[@]+"${args[@]}"}`.

## Common mistakes

- **Looping over files:** not `for f in $(ls *.txt)`, which breaks on spaces and newlines, but `for f in ./*.txt; do [ -e "$f" ] || continue; ...; done`.
- **Reading lines:** not `cat file | while read line`, but `while IFS= read -r line || [ -n "$line" ]; do ...; done < file`. This keeps backslashes and leading spaces, reads a last line that has no newline, and runs in the current shell.
- **Passing found files to a command:** not `find . -name '*.tmp' | xargs rm`, but `find . -name '*.tmp' -exec rm -- {} +` (or `-print0` with `xargs -0`).
- **Changing directory:** `cd -- "$dir" || exit 1` before anything else. With `cd "$dir"; rm -rf ./*`, a failed `cd` deletes the current directory's contents.
- **Printing data:** `printf '%s\n' "$data"`, not `echo "$data"`; `echo` treats `-n`, `-e`, and backslashes differently in every shell.
- **Rewriting a file:** `sort file > "$tmp" && mv -- "$tmp" file`, not `sort file > file`, which empties the file before `sort` reads it.
- **Home directory:** `"$HOME/dir"`, not `"~/dir"`; a tilde inside quotes isn't expanded.
- **Writing a root-owned file:** `printf 'x\n' | sudo tee /etc/file >/dev/null` (with approval); in `sudo echo x > /etc/file` the redirect runs as you.
- **Building commands:** an array, `cmd=(tool --flag "$arg"); "${cmd[@]}"`, never `eval "$cmd $args"`.
- **Reading input:** `read -r var`; without `-r`, backslashes are lost.
- **Remote commands:** `ssh host "rm -- $(printf '%q' "$path")"`; the remote shell parses the string a second time.

## One-liners in agent tool calls

- **Each tool call usually starts a fresh shell.** Variables, `cd`, `export`, and functions don't carry over unless your environment says they do. Use the tool's working-directory option, or chain dependent steps with `&&` in one call.
- **The default shell may be zsh** (macOS), not bash. In zsh: an unquoted `?`, `*`, or `[` that matches no file is an error ("no matches found"), so quote URLs and patterns; a word starting with `=` expands to a command path (`echo ===` fails); unquoted variables don't split into words; `path` is tied to `PATH` and `status` is read-only, so don't use them as variable names. When a snippet needs bash semantics, run it with `bash -c '...'`.
- **Quote paths** with spaces or special characters, every time.
- **Bound anything that might hang** with `timeout` (check `command -v timeout`; elsewhere `perl -e 'alarm shift; exec @ARGV' <seconds> <cmd>`), and disable pagers (`git --no-pager`, `GH_PAGER=cat`, `PAGER=cat`, `--no-pager` flags).
- **Prefer several clear commands to one clever pipeline** when a mistake would be costly. For anything longer than a few lines, write a script file in a temporary directory, lint it, and run it.

## Shell inside other files

- **Makefiles:** each recipe line runs in its own shell, so `cd` doesn't persist between lines; join with `&&` or use `.ONESHELL:`. Write shell variables as `$$var`. For bash with strict mode: `SHELL := bash` and `.SHELLFLAGS := -euo pipefail -c`. `.ONESHELL` and `.SHELLFLAGS` need GNU Make 3.82 or later; macOS's `/usr/bin/make` is 3.81 and ignores both without a warning, so check `make --version`.
- **Dockerfiles:** `RUN` uses `/bin/sh -c`, so a failing command before a pipe goes unnoticed. Use `SHELL ["/bin/bash", "-o", "pipefail", "-c"]` before `RUN` lines with pipes, chain with `&&`, and clean package caches in the same `RUN`.
- **GitHub Actions:** a `run:` step without `shell:` uses `bash -e {0}` on Linux and macOS, without `pipefail`; `shell: bash` adds `-o pipefail`. Never put `${{ github.event... }}` or other user-controlled expressions directly in a `run:` script, because they're pasted in before the shell parses the script (script injection); pass them through `env:` and use `"$VAR"`.
- **Git hooks:** they inherit the environment of whatever ran git, plus `GIT_*` variables. When git runs from a GUI or an IDE, that environment may lack your shell's `PATH` additions. Most hooks run in the repository root, but those that run during a push on the receiving side run in `$GIT_DIR`. Keep hooks fast, give them an explicit shebang (POSIX sh or bash), and make them executable (`chmod +x`).

## Stop and ask when

- A script would delete, overwrite, or move files outside a directory it created, or touch user data.
- It needs `sudo`, changes system configuration, installs software, or edits shell startup files (`~/.zshrc`, `~/.bashrc`, `~/.profile`).
- It downloads and runs code from the internet.
- Making it work on both Linux and macOS would add a dependency (GNU coreutils, a newer bash).
- A destructive command needs `--force` or `--yes`.

## Final report

```
Script:     scripts/rotate-logs.sh (new, executable)
Shell:      bash 3.2+; tested on macOS (bash 3.2.57) and in debian:stable-slim (bash 5.2)
ShellCheck: clean (0.11.0)
Safety:     --dry-run supported; touches only *.log files under the given directory
Ran:        --dry-run on ./tmp/logs (would compress 3 files); not run for real
```

## Tempting shortcuts

| Shortcut | Why not |
|---|---|
| "The variable is always set." | Until a typo, an empty config, or a new caller. `${var:?}` costs nothing. |
| "Quotes aren't needed, there are no spaces." | File names, user input, and paths on other machines will have spaces eventually. |
| "`sed -i` works on my machine." | It fails on macOS (or on Linux, with the macOS form). Use the portable form. |
| "`set -e` will catch it." | Not inside conditions, `local`, or command substitutions. Check what matters explicitly. |
| "The install docs pipe `curl` straight into `sh`." | You'd run unreviewed code with your permissions. Download, read, verify, then run with approval. |
| "I'll add `--force` so it doesn't prompt." | The prompt was there to stop a destructive mistake. Ask the user. |
| "ShellCheck is noisy." | Most of its warnings are real bugs: quoting, unset variables, ignored exit codes. |

## References

| Situation | Read |
|---|---|
| GNU versus BSD (macOS) tool differences, bash 3.2 limits, POSIX sh and dash, detecting the platform | [references/portability.md](references/portability.md) |
| Building blocks: temporary files and cleanup, signal handling, guarded deletes, locks, retries, logging, safe iteration, arrays, in-place edits | [references/patterns.md](references/patterns.md) |
