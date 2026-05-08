---
date: 2026-05-07
tags: [concept, skills, agent-harness]
source: ""
status: developing
---

# Skills

Skills are a way to extend an agent's capabilities with specialized knowledge and workflows.

## What a skill looks like

```
my-skill/
├── SKILL.md      # required: metadata + instructions
├── scripts/      # optional: executable code
├── references/   # optional: documentation
├── assets/       # optional: templates, resources
└── ...           # any additional files or directories
```

A skill is just a folder with a `SKILL.md` at its root. The `SKILL.md` defines the skill's name, description, and instructions for what the skill is supposed to do. Skills can also bundle scripts, references, and templates.

## Why skills exist

Agents are very capable but not always reliable at the work they're supposed to do. Skills patch that gap by giving the agent on-demand context that's user-, domain-, company-, or team-specific.

That gives the agent three things:

- Domain expertise
- Repeatable workflows
- Cross-product reuse

## How agents load skills (progressive disclosure)

1. **Discovery** — at startup, the agent loads only the *name and description* of every skill.
2. **Activation** — when a task matches a skill's description, the agent reads the full `SKILL.md` instructions into context.
3. **Execution** — the agent follows those instructions, including any scripts and references.

The win: an agent can hold a large library of skills without burning context. Only the descriptions live in context until execution time — the heavy details load lazily.

See also: [[MCP]], [[What is Agent Harness]]
