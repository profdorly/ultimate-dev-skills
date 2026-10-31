# Contributing

Reports of where a skill got in your way, let something slip, or quotes a command that no longer works are the most useful contributions. Open an [issue](https://github.com/profdorly/ultimate-dev-skills/issues/new/choose) and include the agent and its version, the tool and its version, and what the agent did. Report security problems privately, as described in [SECURITY.md](SECURITY.md).

## Changing a skill

Every skill follows the same layout:

- **Frontmatter:** `name` (matches the folder), `description`, `license: MIT`, and `metadata.author` and `metadata.version`.
- **Body sections, in order:**
  - a short opening that says why the skill exists;
  - "Rules (non-negotiable)";
  - "Orient first", read-only commands to run before acting;
  - the process steps;
  - "Stop and ask when";
  - "Final report", with an example;
  - "Tempting shortcuts";
  - a "References" table.
- **References:** detail the agent reads only when it needs it, in `references/`, one level deep.
- **LICENSE:** each skill folder has its own, with any attribution.

When you edit one:

1. **Check every command** against the current version of the tool, with its `--help` output or official documentation. Run it when you can. Say which version you checked in the pull request.
2. **Keep it short.** The description loads in every session, so start it with the situations that should trigger the skill and stay under about 500 characters. Keep `SKILL.md` under about 5,000 tokens and move detail into `references/`.
3. **Mind the frontmatter.** An unquoted `: ` inside the description breaks YAML.
4. **Validate:**

   ```bash
   uvx --from skills-ref agentskills validate skills/<name>
   ```

   CI runs the same check on every pull request. Run ShellCheck on new shell snippets.
5. **Bump the version** in the skill's `metadata.version`, following semver: a rule that changes agent behavior is a minor bump, a corrected command is a patch. Add a line to [CHANGELOG.md](CHANGELOG.md).

New skills are welcome when they cover work where an agent's mistake is expensive or hard to undo, and when no existing skill already covers them well. Open an issue to discuss one before writing it.

By contributing, you agree that your contribution is licensed under the [MIT License](LICENSE).
