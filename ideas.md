# Ideas Log

This document captures early concepts for the Project Development Framework. Entries stay informal so we can quickly capture raw feature, architecture, and implementation sparks as they come up.

## Orchestrator Execution

- Start with the orchestrator invoking specialist subagents via command-line calls. Each invocation passes an agent-specific instruction bundle as the system prompt so the subagent knows its role before any conversation begins.
- Later evolve the orchestrator into a hosted managed runtime that can reach out to web-based agents (e.g., Anthropic’s ArCursor) to trigger parallel execution environments for coding or testing. These remote agents may need their own sandbox environments so that deployments or database operations happen safely within the system’s operational boundary. Limited network access or good search tools with tools to download urls for example but not general access to the network.
