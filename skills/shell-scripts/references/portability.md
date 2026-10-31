# Portability: Linux, macOS, and POSIX sh

Linux distributions ship GNU tools; macOS ships BSD-derived tools, and recent macOS releases have added several GNU-style options. When a script must run on both, use the portable form below, or detect the platform and branch. Check behavior on the oldest system the script must support; `command -v <tool>` and a quick test beat assumptions.

## GNU versus BSD tools

| Task | GNU (Linux) | BSD (macOS) | Portable |
|---|---|---|---|
| Edit a file in place | `sed -i 's/a/b/' f` | `sed -i '' 's/a/b/' f` (GNU rejects this form) | `sed -i.bak 's/a/b/' f && rm -- f.bak`, or `perl -pi -e 's/a/b/' f`, or write to a temp file and `mv` |
| Extended regex in sed | `sed -E` (or `-r`) | `sed -E` | `sed -E`; in basic regex, avoid GNU's backslash operators (`\+`, `\?`, and backslash-pipe alternation) |
| Perl-style regex in grep | `grep -P` | not available | `grep -E` with POSIX classes (`[[:space:]]`, `[0-9]`), or `perl -ne 'print if /.../'` |
| Parse a date | `date -d '2026-01-31' +%s` | `date -j -f '%Y-%m-%d %H:%M:%S' '2026-01-31 00:00:00' +%s` (give every field: fields missing from the format are filled from the current time) | `python3 -c` or `perl -MTime::Local`; branch on the OS; set `TZ=UTC` on both for the same result |
| Format an epoch time | `date -u -d @1700000000` | `date -u -r 1700000000` | branch on the OS |
| Current UTC timestamp | `date -u +%Y-%m-%dT%H:%M:%SZ` | same | same (portable) |
| File size in bytes | `stat -c %s f` | `stat -f %z f` | `wc -c < f` |
| File modification time | `stat -c %Y f` | `stat -f %m f` | branch on the OS, or `perl -e 'print((stat shift)[9])' f` |
| Resolve a path | `readlink -f p`, `realpath p` | `readlink -f` (macOS 12.3+), `realpath` (macOS 13+) | for directories, `cd -P -- "$dir" && pwd -P` |
| Skip running with no input | `xargs -r` | `xargs` doesn't run on empty input; `-r` is accepted on current macOS | `xargs -r` on current systems; `find ... -exec ... {} +` avoids the question |
| Custom output from find | `find -printf` | not available | `find ... -exec stat ... {} +` with the right `stat` form, or `-print0` and a loop |
| All lines but the last | `head -n -1` | not available | `sed '$d'` |
| Temporary file or directory | `mktemp`, `mktemp -d` | same | `mktemp -d "${TMPDIR:-/tmp}/name.XXXXXX"`; avoid `mktemp -t`, which means different things on each |
| Base64 without line wrapping | `base64 -w0` | no wrapping by default | pipe `base64` output through `tr -d '\n'` |
| SHA-256 | `sha256sum` | `shasum -a 256` (recent macOS also has `sha256sum`) | `shasum -a 256` where Perl is present, or `openssl dgst -sha256` |
| CPU count | `nproc` | `sysctl -n hw.ncpu` | `getconf _NPROCESSORS_ONLN` |
| Time limit | `timeout 60 cmd` | `timeout` on recent macOS; `gtimeout` from Homebrew coreutils on older | `perl -e 'alarm shift; exec @ARGV' 60 cmd` |
| Lock file | `flock` | not available | `mkdir` as an atomic lock (see [patterns.md](patterns.md#locks)) |
| Copy with parent dirs | `cp --parents`, `install -D` | not available | `mkdir -p -- "$(dirname -- "$dst")" && cp -- "$src" "$dst"` |
| Disk usage in bytes | `du -b` | not available | `du -sk` (kilobytes) |
| Line count | `wc -l < f` prints `2` | prints `       2` (padded) | `$(( $(wc -l < f) ))` or pipe through `tr -d ' '` |
| Creating archives | GNU tar: `--transform`, `--wildcards` | bsdtar: adds `._*` AppleDouble files for extended attributes | `COPYFILE_DISABLE=1 tar -czf ...` on macOS; avoid GNU-only flags |
| awk | gawk or mawk | BWK awk | avoid gawk extensions: `gensub`, `strftime`, `systime`, `asort`, `-i inplace` |
| Relative symlinks | `ln -sr` | not available | compute the relative path yourself, or `cd` into the link's directory first |

Detecting the platform:

```bash
case $(uname -s) in
  Darwin) file_size() { stat -f %z -- "$1"; } ;;
  Linux)  file_size() { stat -c %s -- "$1"; } ;;
  *)      file_size() { wc -c < "$1" | tr -d ' '; } ;;
esac
```

Homebrew's GNU tools use a `g` prefix on macOS (`gsed`, `gdate`, `gtimeout`); a script that needs them should check for them with `command -v` and say how to install them, rather than assume.

## bash 3.2 (stock macOS `/bin/bash`)

`#!/usr/bin/env bash` finds the first `bash` on `PATH`, which on a Mac may be Homebrew's bash 5 or the system's 3.2 depending on the user's setup. If the script must run on stock macOS, don't use:

| Feature | Since | Alternative |
|---|---|---|
| Associative arrays (`declare -A`) | 4.0 | Parallel indexed arrays, `case` lookups, or files |
| `mapfile` / `readarray` | 4.0 | `while IFS= read -r line; do arr+=("$line"); done < file` |
| Case conversion `${v,,}`, `${v^^}` | 4.0 | `tr '[:upper:]' '[:lower:]'` |
| `globstar` (`**`) | 4.0 | `find` |
| `&>>`, and the pipe-and-stderr operator (pipe followed by `&`) | 4.0 | `>> file 2>&1`, and `2>&1` before an ordinary pipe |
| `coproc`, `case` fall-through `;&` and `;;&` | 4.0 | restructure |
| `[[ -v var ]]` | 4.2 | `[ -n "${var+x}" ]` |
| Negative indexes: reading `${arr[-1]}` | 4.2 (assigning and `unset` with a negative index: 4.3) | `${arr[${#arr[@]}-1]}` |
| `declare -n` (namerefs), `wait -n` | 4.3 | pass names and use `eval` carefully, or restructure |
| `${var@Q}` and other transformations | 4.4 | `printf '%q'` |
| Empty `"${arr[@]}"` under `set -u` | fixed in 4.4 | `${arr[@]+"${arr[@]}"}` |
| `$EPOCHSECONDS`, `$EPOCHREALTIME` | 5.0 | `date +%s` |

Also: in bash 3.2, `((count++))` evaluating to 0 doesn't trigger `set -e`, while bash 4+ exits, so behavior differs between machines. Use `count=$((count + 1))`.

## POSIX sh and dash

`/bin/sh` is dash on Debian and Ubuntu (and in many Docker images), BusyBox `ash` on Alpine, and bash in POSIX mode on macOS. A `#!/bin/sh` script can't use:

- arrays (use `"$@"` with `set --` for one list);
- `[[ ... ]]` (use `[ ... ]`), and `==` inside `[ ]` (use `=`);
- `function name {` (use `name() {`);
- brace expansion `{1..10}` (use `seq` or a `while` loop), `<<<` here-strings, `<( )` process substitution;
- `source` (use `.`);
- `$RANDOM`, `read -a`, `echo -e` (use `printf`); `read -p` works in dash and bash but isn't POSIX.

`local` works in dash, ash, and bash but isn't in the POSIX standard. `set -o pipefail` works in bash, zsh, current dash, and BusyBox, but not in every older `sh`; check if you depend on it. In dash, an `EXIT` trap doesn't run when the script is killed by a signal unless you trap the signal too (see [patterns.md](patterns.md#temporary-files-and-cleanup)).

Check a script with `shellcheck --shell=sh` (or `--shell=dash`), and run it with `dash script.sh` to catch bash-isms.

## zsh (the default interactive shell on macOS)

Agents' command tools often run zsh on macOS. Differences from bash that break one-liners:

- An unquoted pattern character (`?`, `*`, `[`) that matches nothing is an error, "no matches found". Quote URLs with query strings and `git` refspecs with `^` or `~` if `EXTENDED_GLOB` is on.
- A word that begins with `=` expands to the path of a command (`=ls` becomes `/bin/ls`), so `echo ====` fails.
- Unquoted variables are not split into words: `v="a b"; for w in $v` loops once.
- Arrays are indexed from 1.
- `path` is an array tied to `PATH`, and `status` is read-only. Don't use them as variable names.
- `echo` interprets backslash escapes by default.

When you need bash behavior in a tool call, run the snippet with `bash -c '...'` (quoting carefully) or write it to a file with a bash shebang.
