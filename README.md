# Ultimate Dev Skills

Skills for coding agents, written in the open [Agent Skills](https://agentskills.io) format. Each skill is a folder under `skills/` with a `SKILL.md` and optional reference files, so it works with any agent that supports the standard: Claude Code, OpenAI Codex, Kiro, Cursor, OpenCode, and others.

## Skills

| Skill | What it does | Version |
|---|---|---|
| [git-workflow](skills/git-workflow/SKILL.md) | Safe local git for coding agents: atomic Conventional Commits, short-lived branches, worktrees, conflict resolution, splitting a messy working tree, recovering work without data loss, and semver releases. Commits, history rewrites, and pushes always need the user's approval. | 0.1.0 |

## Install

With the [skills CLI](https://github.com/vercel-labs/skills), which asks which agents to install into:

```bash
npx skills add profdorly/ultimate-dev-skills --skill git-workflow
```

Or by hand: clone the repo and copy the skill folder into your agent's skills directory.

```bash
git clone https://github.com/profdorly/ultimate-dev-skills.git
mkdir -p ~/.claude/skills
cp -R ultimate-dev-skills/skills/git-workflow ~/.claude/skills/
```

| Agent | User-level directory | Project-level directory |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| OpenAI Codex | `~/.agents/skills/` | `.agents/skills/` |
| Kiro | `~/.kiro/skills/` | `.kiro/skills/` |
| Cursor | `~/.cursor/skills/` | `.agents/skills/` |
| OpenCode | `~/.config/opencode/skills/` | `.agents/skills/` |

Copy the whole folder, not just `SKILL.md`: the skill links to files in its `references/` directory. These paths change between agent versions, so check your agent's docs if a skill doesn't show up.

## Usage

Agents load a skill on their own when a task matches its description (for `git-workflow`: staging, committing, branching, resolving conflicts, recovering work, tagging a release). You can also ask for it by name, for example "use the git-workflow skill".

## Repository layout

```
skills/
  <skill-name>/
    SKILL.md        # frontmatter (name, description, license, metadata.version) + instructions
    references/     # detail the agent reads only when it needs it
    LICENSE         # per-skill license and attribution
```

The `name` in each `SKILL.md` matches its folder name, and each skill carries its own version in `metadata.version`.

## License

MIT, see [LICENSE](LICENSE). Some skills are adapted from other MIT-licensed projects; their attribution is in the skill's own `LICENSE` file (for example, [skills/git-workflow/LICENSE](skills/git-workflow/LICENSE)).
