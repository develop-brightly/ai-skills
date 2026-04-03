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

## Workflows

Workflows are multi-step automation pipelines that go beyond single-agent skills. Place workflow directories in `workflows/`, each containing a `README.md` orchestration script and an `agents/` subdirectory.

### Available Workflows

| Workflow | Description |
|---|---|
| [feature-delivery](./workflows/feature-delivery/) | End-to-end feature delivery using a 6-agent pipeline (PM → Architect → Security + UX → Developer → QA). Produces planning artifacts, working code, and a ship/no-ship QA verdict. |

## Contributing

1. Add new skills to the `skills/` directory
2. Add new workflows to the `workflows/` directory (directory + README pattern)
3. Follow the skill/workflow template format
4. Include a usage example in `examples/`
