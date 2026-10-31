# Shell patterns

Every snippet here works in bash 3.2 and later, and the ones marked POSIX also work in dash. Lint the result with ShellCheck.

## Temporary files and cleanup

```bash
tmp_dir=$(mktemp -d "${TMPDIR:-/tmp}/myscript.XXXXXX") || exit 1
cleanup() { rm -rf -- "$tmp_dir"; }
trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
```

- Create one private directory and put every temporary file inside it; one cleanup removes all of them.
- Bash runs the `EXIT` trap when the script is killed by a signal; dash and some other shells don't. Converting `INT` and `TERM` into `exit` makes the `EXIT` trap run everywhere (POSIX).
- `mktemp` creates the directory with mode 700, so other users can't read or replace your files.

## Guarded deletes

```bash
remove_dir() {
  target=${1:?remove_dir: missing directory}
  case $target in
    / | . | .. | */.. | */../* | "$HOME" | "$HOME"/)
      printf 'refusing to delete %s\n' "$target" >&2
      return 1
      ;;
  esac
  case $target in
    "$work_root"/*) ;;
    *) printf 'refusing to delete %s: outside %s\n' "$target" "$work_root" >&2; return 1 ;;
  esac
  [ -e "$target" ] || return 0
  rm -rf -- "$target"
}
```

- `${1:?message}` stops the function when the argument is unset or empty.
- Restrict deletes to a root the script owns (`work_root`), set once near the top of the script.
- In other commands, `${var:?}` gives the same guarantee inline: `rm -rf -- "${build_dir:?}/"`.

## Locks

`mkdir` is atomic everywhere, so it works as a lock without `flock` (POSIX):

```bash
lock_dir="${TMPDIR:-/tmp}/myjob.lock"
if ! mkdir -- "$lock_dir" 2>/dev/null; then
  printf 'another run is in progress (lock: %s)\n' "$lock_dir" >&2
  exit 1
fi
trap 'rmdir -- "$lock_dir"' EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
```

A run killed with `SIGKILL` leaves the lock behind. Report a stale lock to the user rather than deleting it automatically, unless the script records and checks the owning process ID.

## Retries with backoff

```bash
retry() {
  max_attempts=$1
  shift
  attempt=1
  until "$@"; do
    if [ "$attempt" -ge "$max_attempts" ]; then
      printf 'failed after %s attempts: %s\n' "$attempt" "$*" >&2
      return 1
    fi
    sleep $((attempt * 2))
    attempt=$((attempt + 1))
  done
}

retry 5 curl -fsS --max-time 10 -o "$tmp_dir/data.json" "$url"
```

Only retry operations that are safe to repeat (reads, idempotent writes), and keep a cap on attempts.

## Logging and errors

```bash
log() { printf '%s %s\n' "$(date -u +%Y-%m-%dT%H:%M:%SZ)" "$*" >&2; }
die() { log "error: $*"; exit 1; }
```

Write logs and errors to stderr so that stdout stays clean for data that other programs read.

## Iterating over files

```bash
# Every match, any file name (bash):
find "$dir" -type f -name '*.json' -print0 |
  while IFS= read -r -d '' file; do
    printf 'processing %s\n' "$file"
  done

# Let find run the command (POSIX):
find "$dir" -type f -name '*.json' -exec gzip -- {} +

# Globs in the current directory (POSIX), skipping the literal pattern when nothing matches:
for file in ./*.json; do
  [ -e "$file" ] || continue
  printf 'processing %s\n' "$file"
done
```

The `while` loop on the right of a pipe runs in a subshell, so variables set inside it are lost afterwards. When you need them, read from a process substitution instead: `done < <(find ... -print0)` (bash).

## Reading a file line by line

```bash
while IFS= read -r line || [ -n "$line" ]; do
  printf '[%s]\n' "$line"
done < "$file"
```

`IFS=` keeps leading and trailing spaces, `-r` keeps backslashes, and the `|| [ -n "$line" ]` part handles a last line without a trailing newline (POSIX).

## Building argument lists

```bash
args=(--quiet)
if [ "$verbose" -eq 1 ]; then
  args+=(--verbose)
fi
if [ -n "${config_file:-}" ]; then
  args+=(--config "$config_file")
fi
tool ${args[@]+"${args[@]}"} -- "$input"
```

An array keeps each argument intact, spaces included. The `${args[@]+"${args[@]}"}` form avoids the "unbound variable" error that bash before 4.4 raises for an empty array under `set -u`. In POSIX sh, use `set -- "$@" --verbose` to build the list in the positional parameters.

## Editing a file in place, portably

```bash
tmp_file=$(mktemp "${TMPDIR:-/tmp}/edit.XXXXXX") || exit 1
sed 's/^version = .*/version = "1.4.0"/' "$file" > "$tmp_file"
cat -- "$tmp_file" > "$file"     # keeps the original file's permissions and ownership
rm -f -- "$tmp_file"
```

`mv "$tmp_file" "$file"` is atomic but replaces the file with one that has the temporary file's permissions (mode 600 from `mktemp`); copy the contents back as above, or `chmod` the temporary file to match first.

## The script's own directory

```bash
script_dir=$(cd -P -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)
```

In POSIX sh, use `$0` instead of `${BASH_SOURCE[0]}`; it names the shell rather than the script when the file is sourced.

## Appending a line only once

```bash
# Single quotes on purpose: the line is written to the file literally.
# shellcheck disable=SC2016
line='export PATH="$HOME/.local/bin:$PATH"'
grep -qxF -- "$line" "$file" 2>/dev/null || printf '%s\n' "$line" >> "$file"
```

`-x` matches the whole line and `-F` treats it as a fixed string, so rerunning the script doesn't add duplicates. Editing a user's shell startup files still needs their approval.

## Prompts in scripts that people run

```bash
confirm() {
  if [ "${assume_yes:-0}" -eq 1 ]; then return 0; fi
  if [ ! -t 0 ]; then
    printf '%s: refusing to continue without --yes (no terminal)\n' "$1" >&2
    return 1
  fi
  printf '%s [y/N] ' "$1" >&2
  read -r answer
  [ "$answer" = y ] || [ "$answer" = Y ]
}
```

With no terminal (an agent, CI, a pipe), the prompt fails safe instead of hanging or reading garbage. An agent shouldn't pass `--yes` to such a script unless the user approved the operation it confirms.
