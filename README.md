# bar2133/skills

A curated collection of agent skills for Claude Code, Cursor, and other AI coding agents.

## Installation

### Using `npx skills` (works everywhere)

```bash
npx skills add bar2133/skills -g -y
```

### Cursor

In Cursor Agent chat:

```
/plugin marketplace add bar2133/skills
```

### Claude Code

Register the marketplace:

```
/plugin marketplace add bar2133/skills
```

Then install:

```
/plugin install bar2133-skills@bar2133-skills
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [git-worktree](skills/git-worktree/) | Create and manage git worktrees for single-repo and multi-repo workspaces |

## Adding a New Skill

1. Copy `template/SKILL.md` into a new directory under `skills/`:

```bash
mkdir skills/my-new-skill
cp template/SKILL.md.template skills/my-new-skill/SKILL.md
```

2. Edit the `SKILL.md` frontmatter and instructions:

```yaml
---
name: my-new-skill
description: What this skill does and when the agent should use it.
---
```

3. Update `.claude-plugin/marketplace.json` to include the new skill path in the `skills` array.

4. Update this README's skills table.

5. Commit and push.

## Skill Format

Each skill is a directory under `skills/` containing at minimum a `SKILL.md` file.

The `SKILL.md` uses YAML frontmatter with two required fields:

- **name**: The skill's identifier (should match the directory name)
- **description**: When and why the agent should activate this skill

The body contains markdown instructions the agent follows when the skill is triggered.

Skills can also include additional files (scripts, references, assets) that the `SKILL.md` references.

## License

MIT License - see [LICENSE](LICENSE) for details.
