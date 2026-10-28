# Skills

A collection of reusable Agent Skills, installed and updated with the skills.sh CLI.

## Skills

| Skill | Description |
| --- | --- |
| [flutter-riverpod-architecture](skills/flutter-riverpod-architecture/SKILL.md) | Riverpod 3 layered architecture, code generation, async state, and testing |

## Installation

```sh
# List the skills in this repository
npx skills add ZhuJHua/skills --list

# Install globally and select the skills and agents to use
npx skills add ZhuJHua/skills -g
```

For a project installation, run the same command from the target project and omit `-g`.

## Updates

```sh
# Check installed skills for updates
npx skills check

# Update all global skills
npx skills update -g

# Update a specific global skill
npx skills update <skill-name> -g
```

Maintainers update `skills/<skill-name>/` and push to the installation source branch. Users then fetch the changes with the update command. Installing from a GitHub source lets the CLI record that source and manage subsequent updates.

## Adding a Skill

Create a directory under `skills/` containing a `SKILL.md` with `name` and `description` frontmatter, plus any references, scripts, or assets the skill needs. Keep the directory and skill names stable for installation and updates by name.

References: [Repository layouts and discovery](https://github.com/vercel-labs/skills/blob/main/docs/creating-skills.md), [Installation and update commands](https://github.com/vercel-labs/skills/blob/main/docs/commands.md).
