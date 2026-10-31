# Security policy

These skills are instructions that coding agents follow with your permissions. A security problem here is usually an instruction that could lead an agent to do something unsafe. Examples:

- a command that can destroy data or leak a secret when followed as written;
- a pattern presented as safe that isn't, such as a query, path check, or SSRF filter that can be bypassed;
- a rule that lets an agent skip an approval step it should take.

## Reporting

Please report these privately through [GitHub's private vulnerability reporting](https://github.com/profdorly/ultimate-dev-skills/security/advisories/new), not in a public issue. Include the skill, the file and line, the agent and tool versions, and what an agent could end up doing.

You can expect an acknowledgment within a week. Fixes ship in a new release with a note in the [changelog](CHANGELOG.md).

## Supported versions

Only the latest release gets fixes. Update with `npx skills update`, `gh skill update --all`, or by pulling the repository.

Problems in the agents themselves, or in the skills CLI, belong to those projects. For the skills CLI and skills.sh, see [security.vercel.com](https://security.vercel.com).
