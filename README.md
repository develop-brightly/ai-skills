# AI Skills

A collection of Claude Code skills (slash commands) and AI automation workflows.

## Structure

```
ai-skills/
├── skills/        # Claude Code skill definitions (.md files)
├── workflows/     # Multi-step automation workflows
├── prompts/       # Reusable prompt templates
├── examples/      # Usage examples and sample outputs
└── docs/          # Documentation and guides
```

## Skills

Skills are markdown files that extend Claude Code with custom slash commands. Place skill files in `~/.claude/skills/` or reference them from your project.

See the [skills/](./skills/) directory for available skills.

## Usage

To use a skill in Claude Code, invoke it with `/skill-name` or ask Claude to execute it.

## Contributing

1. Add new skills to the `skills/` directory
2. Follow the skill template format
3. Include a usage example in `examples/`
