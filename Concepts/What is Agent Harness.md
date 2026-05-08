---
date: 2026-05-07
tags: [concept, agent-harness, core]
source: ""
status: developing
---

# What is Agent Harness

An LLM is just a text generator. What turns it into an agent is the harness around it.

> Agent = LLM model + Harness

The harness is the software infrastructure that surrounds the model. It manages lifecycle, memory, and external tool usage — in short, it's what makes the model autonomous and useful.

The model holds the intelligence. The harness is what makes it act on the world.

```
Model       != Agent
Model + (code + configuration + execution logic) = Agent
```

## What a harness is made of

```
Harness = state management
        + tool execution
        + feedback loops
        + enforceable constraints
```

Concretely, that means:

- [[System Prompts]]
- Tools and [[MCP]]s
- [[Skills]]
- Bundled infrastructure and orchestration logic
- Hooks/middleware for deterministic execution

## Why this matters

On their own, LLMs only take input (text, audio, video) and emit text. They can't:

- Maintain state across a conversation
- Run code
- Search the web
- Call APIs

None of that lives inside the model. All of it lives in the harness.
