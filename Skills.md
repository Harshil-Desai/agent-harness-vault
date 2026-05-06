Skills are a way to extend AI agents' capabilities with specialized knowledge and workflows.

my-skill/
├── SKILL.md          # Required: metadata + instructions
├── scripts/          # Optional: executable code
├── references/       # Optional: documentation
├── assets/           # Optional: templates, resources
└── ...               # Any additional files or directories

this is how a skill looks, a folder with SKILL.md file, which has name, description and instruction of what that skill is supposed to do. Skills can also use scripts, references and templates.

Agents are very capable, but sometimes not reliable to do the work they are supposed to do, to remove this limitations skills are used. Skills consists user-specific, domain-specific, company-specific, team-specific context, which can be loaded by agents on demand. 

It gives agents domain expertise, repeatable workflows and cross product reuse.

Agents load skills through something called progressive disclosure:
1. Discovery: Agent loads only name and description of all skills at startup.
2. Activation: When a task matches a skill's description, then agent reads SKILL.md file's instructions into context.
3. Execution: Agent follows instructions (that includes the scripts and references as well)

So agent can hold multiple skills into the context without using much context as it only has to keep description of skill until the time of execution.

