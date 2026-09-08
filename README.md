# skills

This repository is the **NeatPlatform plugin marketplace**.
It hosts reusable skills, commands, agents, and hooks, packaged as plugins, so
individuals and consumer repositories can install only what is relevant to their work.

Skills are grouped into plugins by functionality and consumers only pull in what they need.

## Quick Start

```
/plugin marketplace add neatplatform/skills   # Add the marketplace once
/plugin marketplace list neatplatform-skills  # Browse everything available
/plugin install dev@neatplatform-skills       # Install plugin you need
```

## Available Plugins

| Plugin  | Description                                                            |
|---------|------------------------------------------------------------------------|
| `dev`   | Developer skills for coding, testing, and troubleshooting workflows    |
| `infra` | Infrastructure skills for provisioning and maintaining cloud resources |

## Adding New Skills

Skills live under a plugin's `skills/` directory, one subdirectory per skill:
`plugins/<plugin>/skills/<skill-name>/SKILL.md`.

  1. Pick the plugin that best matches the skill's functional area
     (`dev`, `infra`, etc.), or propose a new plugin if none fit.
  2. Create a new directory under that plugin's `skills/` folder,
     named after the skill, containing a `SKILL.md`.
  3. Write `SKILL.md` with frontmatter (`name` and a `description`
     that states what the skill does and when to use it, since that's what Claude matches against)
     followed by the step-by-step instructions the skill should follow.
  4. Bump the `version` field in that plugin's `.claude-plugin/plugin.json` (semver),
     so consumers know the plugin changed and pick up the update.
  5. Validate the skill with `claude plugin eval` before submitting it for review.

## Resources

  - [Create and distribute a plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
  - [Manage plugins for your organization](https://support.claude.com/en/articles/13837433-manage-plugins-for-your-organization)
