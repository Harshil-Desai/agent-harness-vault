An LLM model is just a text generator. What makes it an agent is Harness.
Agent = LLM model + Harness

Agent Harness is software infrastructure which surrounds the LLM model, it manages its lifecycle, memory management, external tool usage, in short makes it autonomous.

The model contains the intelligence and a harness is what makes it useful.

Model != Agent but
Model + (Code + configuration + execution logic) = Agent

Harness = State management + tool execution + feedback loops + enforceable constraints
Harness = [[System prompts]] + Tools + [[Skills]] + [[MCP]]s + - Bundled Infrastructure, Orchestration Logic + Hooks/Middleware for deterministic execution

All LLMs can do is take data (text, video, audio etc.) and output text. That's it. It cannot maintain states for whole conversation, it cannot run a code, it cannot do web search, none of these fall under LLMs capabilities

