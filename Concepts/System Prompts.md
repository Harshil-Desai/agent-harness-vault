---
date: 2026-05-07
tags: [concept, prompts, agent-harness]
source: ""
status: developing
---

# System Prompts

System prompts are instructions given to the LLM *before* it ever interacts with the user. Think of them as the model's global context — the rules and frame it carries into every conversation.

## What they're used for

- Security rules and safety fallbacks
- Improving accuracy and mitigating risk before user input arrives
- Giving the LLM a persona or character to follow
- Bounding the LLM into rules and constraints
- Providing domain knowledge and context
- Defining rubrics or frameworks the LLM uses to assess and evaluate

In short: system prompts are the standing instructions the LLM follows in every conversation. They're the harness's first lever for shaping behaviour.

See also: [[User Prompts]], [[What is Agent Harness]]
