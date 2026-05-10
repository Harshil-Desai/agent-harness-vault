---
date: 2026-05-07
tags: [concept, mcp, agent-harness]
source: ""
status: fleeting
---

# MCP (Model Context Protocol)

### What MCP is:
MCP(model context protocol) is standard open source way to connect AI applications to external tools/systems. 

We can use MCP to connect our AI application to connect to outer data source, tools or workflows.

![[MCP_Components.excalidraw]]

MCP server is just the program that serves context data, no matter from where, it can be local, it can be remote, can be from a filesystem, can be from an application/website.

It has two layers:
1. Data Layer: It defines [[JSON-RPC]] based protocol for client-server communication + lifecycle management + core primitives (tools, prompts, resources etc.)
2. Transport Layer: It defines the communication mechanisms and channels that enable data exchange between clients and servers. 
Data layer is the inner layer and transport layer is the outer layer.

Transport layer works on 2 transport methods:
1. STDIO transport: used when communication is between local processes on the same machine.  
2. Streamable HTTP transport: used when communication is needed remotely

| MCP                                      | Skills                                                                                                                                               | Tools                                                                                        |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Connect AI Application to external tools | Natural language instructions, workflow steps and best practices to guide the agent on how to solve a specific problem or how to use a specific tool | Hardcoded function definitions that are called by an LLM to directly execute a specific task |
### Schema Injection Cost
If we have configured some MCPs, we have to bear schema injection cost which is token cost caused because of loading JSON schema of MCPs in every prompt. Most frameworks unconditionally inject all registered tool schemas, wasting tokens when only 3–5 tools are used. 

See also: [[What is Agent Harness]], [[Skills]]
