# Skills

Agent Skills for Flutter development, installed and updated with the skills.sh CLI.

## Skills

| Skill | Description |
| --- | --- |
| [flutter-riverpod-architecture](skills/flutter-riverpod-architecture/SKILL.md) | Riverpod 3 layered architecture, code generation, async state, and testing |

## Installation

```sh
# List the skills in this repository
npx skills add ZhuJHua/skills --list

# Install the selected skill globally for Codex
npx skills add ZhuJHua/skills --skill flutter-riverpod-architecture -a codex -g
```

For a project installation, run the same command from the target project and omit `-g`.

## Updates

```sh
# Check installed skills for updates
npx skills check

# Update this global skill
npx skills update flutter-riverpod-architecture -g

# Update all global skills
npx skills update -g
```

Maintainers update `skills/<skill-name>/` and push to the installation source branch. Users then fetch the changes with the update command. Installing from a GitHub source lets the CLI record that source and manage subsequent updates.

## Adding a Skill

Create a directory under `skills/` containing a `SKILL.md` with `name` and `description` frontmatter, plus any references, scripts, or assets the skill needs. Keep the directory and skill names stable for installation and updates by name.

References: [Repository layouts and discovery](https://github.com/vercel-labs/skills/blob/main/docs/creating-skills.md), [Installation and update commands](https://github.com/vercel-labs/skills/blob/main/docs/commands.md).
