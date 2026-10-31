# Ultimate Dev Skills

[![Validate skills](https://github.com/profdorly/ultimate-dev-skills/actions/workflows/validate.yml/badge.svg)](https://github.com/profdorly/ultimate-dev-skills/actions/workflows/validate.yml)
[![skills.sh](https://skills.sh/b/profdorly/ultimate-dev-skills)](https://skills.sh/profdorly/ultimate-dev-skills)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Skills for coding agents that cover the work where an agent's mistake is expensive or hard to undo: commits and pushes, pull requests and CI, database migrations, new dependencies, and shell scripts. Four more skills cover debugging, testing, code review, and secure coding.

The skills use the open [Agent Skills](https://agentskills.io) format. That means they work in Claude Code, OpenAI Codex, Cursor, Gemini CLI, GitHub Copilot, Kiro, OpenCode, Factory Droid, Devin Desktop (formerly Windsurf), and any other agent that supports the standard.

- [Why](#why)
- [The skills](#the-skills)
- [Install](#install)
- [Using the skills](#using-the-skills)
- [Context cost](#context-cost)
- [How the skills are checked](#how-the-skills-are-checked)
- [Status and feedback](#status-and-feedback)

## Why

Left to their defaults, coding agents do things like this:

- sweep a `.env` file into `git add -A`, then answer a rejected push with `--force`;
- install a package name they made up, which an attacker may have registered;
- run `prisma migrate reset` or `db push` without checking which database `DATABASE_URL` points to;
- delete or skip a failing test, or add `continue-on-error`, to get CI green;
- write `sed -i` that works on Linux and breaks on macOS;
- hang on a command that opens an editor, a pager, or a watch mode.

Each skill gives the agent rules for one area and the exact commands to follow them:

- **Approval before anything hard to undo.** The agent proposes commits, pushes, merges, destructive migrations, and new dependencies, then waits for your OK. Read-only commands run freely.
- **Exact, current commands.** The commands come with the flags that make them safe and non-interactive. They were checked against the tool versions in use today, such as gh 2.103, npm 11, pnpm 12, uv 0.13, and Postgres 18 (see [How the skills are checked](#how-the-skills-are-checked)).
- **Written for agents.** Nothing waits for keyboard input. Long output goes to a file instead of the conversation, and the agent reports only what it ran and saw.

## The skills

**Operations that are hard to undo**

| Skill | What it stops | Use it for |
|---|---|---|
| [git-workflow](#git-workflow) | Commits without asking, secrets in commits, force pushes, lost work | Staging, commits, branches, rebases, conflicts, recovery, releases |
| [pull-requests](#pull-requests) | Undescribed PRs, blind CI re-runs, skipped checks, unguarded merges | Opening PRs, fixing failed CI, review threads, merging, with the `gh` CLI |
| [database-migrations](#database-migrations) | Resets against the wrong database, table locks, renames that drop data | Schema changes, backfills, rollbacks, drift, in 17 migration tools |
| [dependencies](#dependencies) | Hallucinated or typosquatted packages, disabled supply-chain protections, mass upgrades | Adding, upgrading, removing, and auditing packages in 14 package managers |
| [shell-scripts](#shell-scripts) | Unquoted deletes, scripts that break on macOS, commands that hang | Scripts, one-liners, Makefiles, Dockerfiles, CI steps, git hooks |

**Engineering discipline**

| Skill | What it stops | Use it for |
|---|---|---|
| [debugging](#debugging) | Symptom patches, swallowed errors, sleeps in races, fixes nobody verified | Bugs, crashes, regressions, failures only in CI or production, hangs |
| [testing](#testing) | Deleted, skipped, or weakened tests; blind snapshot updates; watch modes that never exit | Writing and running tests, flaky tests, snapshots, mocks, coverage |
| [code-review](#code-review) | Unverified "bugs", nit floods, instructions hidden in a PR | Reviewing diffs, branches, and PRs; self-review; answering feedback |
| [secure-coding](#secure-coding) | Disabled TLS checks, string-built queries, missing ownership checks | Endpoints, auth, uploads, webhooks, secrets, crypto, LLM features |

Each skill works on its own. When a task crosses into another skill's area, the skill names the other one (debugging hands flaky tests to testing, for example), so they work best installed together.

### git-workflow

Safe local git: the agent proposes each commit and waits.

- **Approval first.** Commits, merges, rebases, tags, and pushes wait for your OK.
- **Nothing gets lost.** Before anything that can discard work, the agent shows what would be lost and offers a reversible alternative: a snapshot, a backup branch, or `reset --keep`.
- **Clean history.** Atomic Conventional Commits, and a recipe for splitting a messy working tree into separate commits without interactive staging.
- **The hard cases.** Merge and rebase conflicts (lockfiles included), lost commits and dropped stashes, a committed secret, `bisect`, worktrees, and releases with tags and changelogs.

Before committing, the agent shows a proposal like this:

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

Try: "commit this", "split these changes into separate commits", "the rebase stopped with conflicts, sort it out", "I deleted a branch by mistake, get it back", "release 1.3.0". Requires git 2.31 or later and a POSIX shell (Git Bash or WSL on Windows).

### pull-requests

GitHub pull requests and CI with the `gh` CLI.

- **Every write to GitHub waits for approval:** creating, editing, commenting, re-running, merging.
- **Draft PRs with evidence.** The description has a summary, the tests that actually ran, and the risks, and follows the repository's template.
- **CI failures fixed at the cause.** The agent finds the failing step with `gh run view --log-failed`, decides whether the change caused it, and reproduces it locally. Skipping checks, weakening workflows, and blind re-runs are off the table.
- **Review threads.** The agent lists unresolved threads, checks each comment against the code, and proposes replies.
- **Guarded merges** with the repository's merge method and `--match-head-commit`. Never `--admin`.

Try: "open a PR for this", "CI is red, fix it", "answer the review comments", "merge it". Requires the GitHub CLI, signed in. GitHub only.

### database-migrations

Schema and data changes that don't lock tables or lose data.

- **Checks the target first.** The agent prints the host and database a command will hit (never the password) and asks before touching anything that isn't a local, disposable database.
- **No destructive commands without approval:** resets, drops, `db push`, and `--force`. It never sets safety overrides such as Prisma's AI-consent variable on its own.
- **Reviews generated migrations.** Generators often turn a rename into a drop and an add, which loses the column's data.
- **Lock-safe patterns for Postgres and MySQL:** expand and contract, `lock_timeout`, concurrent indexes, `NOT VALID` constraints, and batched backfills. SQLite gets its table-rebuild procedure.
- **Commands for 17 tools.** Status, dry-run, apply, and rollback for Prisma, Django, Rails, Alembic, Drizzle, Knex, TypeORM, Sequelize, Flyway, Liquibase, golang-migrate, goose, sqlx, Diesel, Ecto, EF Core, and Laravel, plus the commands in each that destroy data.

Try: "add a NOT NULL column to invoices", "rename this column", "the migration failed halfway, fix it".

### dependencies

Agents invent package names, and attackers register them. This skill checks a package before anything gets installed.

- **Checks the package first:**
  - registry lookups for npm, PyPI, crates.io, Go, RubyGems, Packagist, and NuGet;
  - a list of red flags, such as a brand-new package, few downloads, a near-miss name, or install scripts;
  - an OSV vulnerability check;
  - reading the install scripts without running them.
- **Uses the project's own package manager and lockfile:** npm, pnpm, Yarn, Bun, uv, Poetry, pip, Cargo, Go, Bundler, Composer, Maven, Gradle, and NuGet.
- **Keeps supply-chain protections on:**
  - release-age delays (pnpm `minimumReleaseAge`, npm `min-release-age`);
  - install-script approval, one package at a time (npm `allowScripts`, pnpm `allowBuilds`, Bun `trustedDependencies`);
  - frozen lockfiles.
- **One upgrade at a time,** after reading the changelog. Vulnerability alerts get the smallest fix that works.

Try: "add a library for parsing dates", "upgrade React to the next major", "fix this npm audit alert".

### shell-scripts

Shell that behaves the same on Linux and macOS.

- **Safe by default:** quoting, `set -euo pipefail` and the places it doesn't protect you, guarded `rm -rf`, temporary directories with cleanup traps, and `--dry-run`.
- **Portable:**
  - a GNU versus BSD table (`sed -i`, `date`, `stat`, `mktemp`, `xargs`);
  - the limits of macOS's bash 3.2;
  - POSIX sh and dash;
  - the zsh behaviors that break agents' one-liners.
- **Shell inside other files:** Makefiles, Dockerfiles, GitHub Actions `run:` steps (including script injection), and git hooks.
- **Linted.** The agent runs ShellCheck on what it writes. The skill's own snippets pass ShellCheck, and its script template runs on bash 3.2 and 5.

Try: "write a script to rotate these logs", "this works on Linux but not on my Mac".

### debugging

Root-cause debugging instead of patching the first plausible cause.

- **Reproduce first,** with the narrowest fast command, and turn the reproduction into a failing test.
- **One hypothesis at a time,** with a short log of what was ruled out. After three failed fixes, the agent stops, lists what it has ruled out, and checks with you before trying again.
- **Fix the cause** at the right layer, look for the same mistake elsewhere, and prove the regression test fails without the fix.
- **The hard cases:**
  - failures that won't reproduce;
  - intermittent failures, including how many clean runs make "fixed" credible;
  - failures that only happen in CI or in production;
  - hangs, memory growth, and races;
  - `git bisect run` in a throwaway worktree.

Try: "this endpoint returns the wrong page", "it only fails in CI", "the tests hang".

### testing

Tests that can fail, run without hanging, and never get weakened to pass.

- **Runs the project's real test command** without interactive or watch modes. It has notes for about twenty runners (`vitest run`, `PLAYWRIGHT_HTML_OPEN=never`, `go test -count=1`) and lists the name filters that quietly run zero tests.
- **Never weakens a test** to get a green run: no skips, loosened assertions, blind snapshot updates, or retries.
- **Every new test must fail once** before the agent counts it, and tests stay deterministic: time, randomness, network, files, and environment are controlled, with tools listed for each language.
- **Flaky tests** are diagnosed by changing one factor at a time instead of retrying them away.

Try: "add tests for this", "this test is flaky", "update the snapshots".

### code-review

Reviews built on verified findings, not guesses.

- **Reads the change in context** (callers, deleted code) and runs it before judging it.
- **Only verified findings,** each with file:line, a concrete scenario that triggers it, and a suggested fix. They're ranked blocker, should fix, nit, or question, and end in a verdict. Nothing the linter already enforces.
- **Treats the PR as untrusted input.** Instructions inside a diff or description get reviewed, not followed, and code from forks doesn't run without approval.
- **Self-review** of the agent's own diff before it reports a task done, and a process for answering review feedback on its merits.

Try: "review this branch", "review PR 231", "check your changes before you finish".

### secure-coding

Safe defaults wherever code crosses a trust boundary.

- **Never disables a security control to make something work:** TLS verification, CSRF protection, auth middleware, signature checks, CORS.
- **Safe patterns per language** for SQL, shell commands, file paths, SSRF, XSS, templates, deserialization, XML, open redirects, and mass assignment. The path-traversal snippets were run in Python, Node, Go, and Java, and the Go SSRF dialer was tested against loopback, private, cloud-metadata, and IPv4-mapped addresses.
- **Authentication and authorization:**
  - object-level checks in the query;
  - session handling;
  - current OWASP password-hashing parameters;
  - JWT validation, OAuth with PKCE, password resets, and webhook signatures;
  - secrets kept out of public variable prefixes such as `NEXT_PUBLIC_`.
- **A five-line threat model** before any change that crosses a trust boundary.

Try: "add a file upload endpoint", "is this login safe?", "threat-model this feature".

## Install

**Requirements:** git and a POSIX shell (on Windows, Git Bash or WSL). pull-requests needs the [GitHub CLI](https://cli.github.com), signed in. Some lookups in dependencies use `curl` and `jq`, and a few optional recipes use Docker.

### Recommended: one command for your agent

Run it from your project's root. It installs every skill for that project; add `-g` to install them for all your projects instead.

| Agent | Command |
|---|---|
| Claude Code | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a claude-code` |
| OpenAI Codex | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a codex` |
| Cursor | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a cursor` |
| Gemini CLI | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a gemini-cli` |
| GitHub Copilot | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a github-copilot` |
| Kiro (CLI and IDE) | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a kiro-cli` |
| OpenCode | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a opencode` |
| Factory Droid | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a droid` |
| Devin Desktop (formerly Windsurf) | `npx skills add profdorly/ultimate-dev-skills --skill '*' -a devin` |

- **Several agents:** repeat `-a`, as in `-a claude-code -a codex`.
- **One skill:** use its name instead of `'*'`, as in `--skill dependencies`.
- **See the list first:** `npx skills add profdorly/ultimate-dev-skills --list`.
- **If you ask an agent to run the install,** add `-y`. Agents can't answer the confirmation prompts, so without `-y` the command waits forever. With it, you're skipping the review step, so read the skills first.

The command uses the [skills CLI](https://github.com/vercel-labs/skills), which needs Node.js 22.20 or later.

- **Where the files go:** with one agent, the CLI copies the skills into that agent's folder. With several, it keeps one copy in `.agents/skills/` and links the other agents' folders to it. It prints what it did. The links are relative, so they work for teammates too.
- **Sharing with a team:** the CLI records the source in `skills-lock.json`. Commit that file and the skill folders. On teams with Windows users, add `--copy` to install plain copies instead of links.
- **Other commands:** `npx skills list` shows what's installed, `npx skills check` looks for updates, `npx skills update` installs them, and `npx skills remove <name>` uninstalls a skill.

### Other installers

**Claude Code plugin.** In Claude Code 2.1.275 or later, one command at the prompt:

```
/plugin install ultimate-dev-skills --marketplace profdorly/ultimate-dev-skills
```

In older versions, add the marketplace first:

```
/plugin marketplace add profdorly/ultimate-dev-skills
/plugin install ultimate-dev-skills@ultimate-dev-skills
```

Plugin skills are namespaced, so you call them as `/ultimate-dev-skills:git-workflow`. Updates come through `/plugin`. Use either the plugin or `npx skills add` for Claude Code, not both; with both, every skill loads twice.

**GitHub CLI** (`gh skill` is in preview):

```bash
gh skill install profdorly/ultimate-dev-skills --all --agent claude-code --scope user
gh skill install profdorly/ultimate-dev-skills database-migrations --agent github-copilot
gh skill update --all
```

`--agent` takes mostly the same names as the skills CLI; for Devin Desktop use `devin`. `--scope` is `project` (the default) or `user`, and `--pin <tag>` installs a fixed version.

**Gemini CLI:**

```bash
gemini skills install https://github.com/profdorly/ultimate-dev-skills.git --path skills
gemini skills install https://github.com/profdorly/ultimate-dev-skills.git --path skills/debugging --scope workspace
```

The first line installs all skills for your user, the second one skill for the current project. Keep `--path`; without it the installer finds no skills in this repository. Gemini asks you to confirm the install, or skips the question with `--consent`, and asks again when a skill first activates.

**GitHub Copilot CLI:** `copilot skill add <directory or URL>` adds a skill folder. Inside a session, `/skills reload` picks up new skills without a restart.

**OpenAI Codex:** the built-in `$skill-installer` fetches skills from a repository when you ask it to.

**Kiro IDE:** open **Agent Steering & Skills**, choose **+**, then **Import a skill**, then **GitHub**. Paste a skill folder's URL, such as `https://github.com/profdorly/ultimate-dev-skills/tree/main/skills/dependencies`. Each import adds one skill.

### Where each agent looks

Most agents read the shared `.agents/skills/` folder in the project and `~/.agents/skills/` in your home directory. Claude Code and Kiro use their own folders.

| Agent | Project folder | All projects | Call a skill by name | See what loaded |
|---|---|---|---|---|
| Claude Code | `.claude/skills/` | `~/.claude/skills/` | `/dependencies` | `/skills` |
| OpenAI Codex (CLI, IDE, ChatGPT desktop) | `.agents/skills/` | `~/.agents/skills/` | `$dependencies` | `/skills` |
| Cursor | `.agents/skills/` or `.cursor/skills/` | `~/.cursor/skills/` or `~/.agents/skills/` | type `/` in Agent chat | Customize → Skills |
| Gemini CLI | `.agents/skills/` or `.gemini/skills/` | `~/.agents/skills/` or `~/.gemini/skills/` | ask for it by name | `/skills list` |
| GitHub Copilot (CLI, VS Code) | `.github/skills/` or `.agents/skills/` | `~/.copilot/skills/` or `~/.agents/skills/` | `/dependencies` in your prompt | `/skills list` (CLI) |
| Kiro (CLI and IDE) | `.kiro/skills/` | `~/.kiro/skills/` | `/dependencies` | `/context show` (CLI); Agent Steering & Skills panel (IDE) |
| OpenCode | `.agents/skills/` or `.opencode/skills/` | `~/.agents/skills/` or `~/.config/opencode/skills/` | ask for it by name | `opencode debug skill` (terminal) |
| Factory Droid | `.agents/skills/` or `.factory/skills/` | `~/.agents/skills/` or `~/.factory/skills/` | `/dependencies` | `/skills` |
| Devin Desktop (formerly Windsurf) | `.agents/skills/` or `.devin/skills/` | `~/.agents/skills/` or `~/.config/devin/skills/` | `/dependencies` | Customize → Skills |

These paths come from each agent's documentation as of October 2026. Agents move them between versions; if a skill doesn't show up, check the agent's docs and start a new session.

A few agents have conditions on loading skills:

- **Cursor and OpenCode** also read Claude Code's folders (`.claude/skills/` and `~/.claude/skills/`). Copilot reads `.claude/skills/` in the project. So one Claude Code install also reaches those agents.
- **Kiro:**
  - The default agent loads both folders automatically. In Kiro CLI 2.33 and later, project skills load only after you choose "Trust, continue" for the workspace.
  - A custom agent needs `"resources": ["skill://.kiro/skills/**/SKILL.md", "skill://~/.kiro/skills/**/SKILL.md"]` in `.kiro/agents/<agent>.json`.
- **Copilot cloud agent:** it doesn't see personal skills on your machine. Put them in `.copilot/skills/` in your dotfiles repository, or commit them to the project.
- **Devin Desktop:** agents are off in workspaces opened in Restricted Mode, so skills don't load there either.

### Install by hand

Copy whole folders, not just `SKILL.md`; the skills read their `references/` files.

```bash
git clone https://github.com/profdorly/ultimate-dev-skills.git

# Codex, Cursor, Gemini CLI, Copilot, OpenCode, Droid, Devin Desktop:
mkdir -p ~/.agents/skills && cp -R ultimate-dev-skills/skills/. ~/.agents/skills/
# Claude Code:
mkdir -p ~/.claude/skills && cp -R ultimate-dev-skills/skills/. ~/.claude/skills/
# Kiro:
mkdir -p ~/.kiro/skills && cp -R ultimate-dev-skills/skills/. ~/.kiro/skills/
```

For a single project, use the project folder from the table instead of the home-directory one. For a single skill, copy `ultimate-dev-skills/skills/<name>`.

To get updates with `git pull`, link the folders instead of copying them:

```bash
git clone https://github.com/profdorly/ultimate-dev-skills.git ~/ultimate-dev-skills
mkdir -p ~/.agents/skills
for skill in ~/ultimate-dev-skills/skills/*/; do
  ln -s "${skill%/}" ~/.agents/skills/
done
```

These links use absolute paths on your machine, so don't commit them to a project; list them in `.git/info/exclude` instead. `rm ~/.agents/skills/<name>` (no trailing slash) removes a link and leaves the clone untouched. On Windows, symlinks need Developer Mode or WSL.

## Using the skills

You don't need to do anything special. At the start of a session, the agent sees each skill's name and a short description. When a task matches a description, it loads that skill. Write your request the way you normally would; each skill section above has examples under "Try".

If the agent handles a task without the skill (a "quick commit" is a common case), name the skill: "use the git-workflow skill", or call it directly with `/git-workflow` in Claude Code, Cursor, Copilot, Kiro, Droid, and Devin Desktop, and `$git-workflow` in Codex.

The skills ask before acting in the situations listed in their "Stop and ask when" sections. Expect proposals like the commit one above before commits, pushes, PRs, merges, destructive migrations, and new dependencies. A direct request such as "commit everything" counts as approval for that one commit when nothing in the tree needs your attention; otherwise the agent proposes a plan first.

## Context cost

Skills load in stages, so installing all nine costs little until a skill is actually used. Measured with the `o200k_base` tokenizer; other models count a little differently.

| Loaded | When | Size |
|---|---|---|
| Name and description of every skill | At the start of every session | About 830 tokens for all nine |
| A skill's `SKILL.md` | When a task matches it | 2,100 to 3,100 tokens (git-workflow: 5,000) |
| One reference file | Only when the skill sends the agent there | 1,200 to 4,500 tokens |

Most descriptions start with the situations that should trigger them, so they still match if an agent shortens them to fit its skill-list budget (Codex, for example, does this when many skills are installed).

## How the skills are checked

As of October 2026:

- **Commands and flags** were checked against the installed tools' help output or official documentation: gh 2.103, npm 11, pnpm 12, Bun 1.4, uv 0.13, pytest, Jest, Vitest, Playwright, and others.
- **Recipes that can run were run:**
  - the Postgres patterns against Postgres 18;
  - the online-DDL rules against MySQL 8.4;
  - the SQLite `ALTER TABLE` claims against SQLite 3.54;
  - the path-traversal snippets in Python, Node, Go 1.25, and Java 25, and the Go SSRF dialer;
  - the shell snippets on macOS's bash 3.2 and on Linux bash 5.2, and through ShellCheck 0.11;
  - the install commands above, in throwaway projects.
- **Format:** every skill passes `agentskills validate` from the reference [skills-ref](https://github.com/agentskills/agentskills) tools.
- **Independent fact-check:** separate reviewers rechecked every skill against documentation and tests. That review found and fixed 24 errors before release.

There are no behavioral evaluations yet, meaning runs that compare an agent with and without a skill. That's the next step.

## Status and feedback

Version 0.1.0 for every skill; see the [changelog](CHANGELOG.md). git-workflow has been used on a few real projects; the others are new. Tool versions move fast, so some details will go stale. If a skill got in your way, let something slip, or quotes a command that no longer works, please open an [issue](https://github.com/profdorly/ultimate-dev-skills/issues/new/choose) with the agent, the tool version, and what happened. Security problems go through [private reporting](SECURITY.md), and changes to skills follow [CONTRIBUTING.md](CONTRIBUTING.md).

## Repository layout

```
skills/
  <skill-name>/
    SKILL.md        # frontmatter (name, description, license, metadata.version) and instructions
    references/     # detail the agent reads only when it needs it
    LICENSE         # per-skill license and attribution
.claude-plugin/     # Claude Code plugin and marketplace manifests
.github/            # CI that validates every skill, and the issue form
```

The `name` in each `SKILL.md` matches its folder name, and each skill carries its own version in `metadata.version`.

## License

MIT, see [LICENSE](LICENSE). Some skills are adapted from or cite other projects; their attribution is in the skill's own `LICENSE` file (for example, [skills/git-workflow/LICENSE](skills/git-workflow/LICENSE) and [skills/secure-coding/LICENSE](skills/secure-coding/LICENSE)).
