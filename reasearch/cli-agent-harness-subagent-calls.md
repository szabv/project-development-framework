# Research: Calling Subagents via CLI in Target Harnesses

Scope: how to invoke subagents (LLM workers) from the main orchestrator when running inside common agent harnesses. Harnesses considered: Claude Code (Anthropic IDE harness), Codex CLI (this project harness), Gemini CLI (Google).

## Sources Consulted
- Anthropic Claude Code overview: https://docs.anthropic.com/claude/docs/claude-code
- Anthropic tool-calling docs: https://docs.anthropic.com/claude/docs/tools
- Gemini CLI GitHub repo: https://github.com/google-gemini/gemini-cli
- Gemini CLI documentation: https://geminicli.com/docs/

## Claude Code (Anthropic IDE Harness)
- Environment: Claude-powered IDE experience with integrated editor and terminal.
- Shell behavior:
  - Commands run in a project workspace terminal and stream results back into the conversation.
  - Suitable for orchestrator patterns where Claude issues CLI calls to other agents or tools.
- Prompting considerations for subagent calls:
  - Use a short “plan → execute” loop: first outline which subagent calls will be made, then run them.
  - Be explicit about files and directories, and ask for confirmation before commands that modify code, run migrations, or deploy.
  - Keep instructions concise but structured so the terminal and editor actions are easy to follow.
- Subagent call pattern from Claude Code:
  - Write the subagent’s XML system prompt to a temp file in the workspace (for example, `tmp/subagent.xml`).
  - Write the task payload (feature or step-specific instructions) to a second file (for example, `tmp/task.txt`).
  - Run a CLI such as `gemini` or another orchestrated agent with flags that accept system/prompt files and input payloads.
  - After execution, summarize what happened, including any generated files or diffs, and decide whether to proceed or adjust.
- Safety:
  - Avoid destructive git commands; prefer incremental changes and clear diffs.
  - Ask before touching database, deployment scripts, or credentials.

## Codex CLI
- Environment: terminal-oriented harness where the orchestrator has direct shell and file access.
- Tooling conventions:
  - Use `rg` for search and `apply_patch` for targeted file edits.
  - Maintain a lightweight plan with clear step transitions; keep user-facing explanations concise.
- Subagent call pattern:
  - Emit XML system prompt for the subagent into a file (for example, `spec/prompts/<agent-name>-system.xml`).
  - Emit task payload (goal, context excerpt, constraints, success criteria) into a second file.
  - Invoke the subagent process or model using CLI flags that accept a system prompt and input payload; prefer passing file contents or `@file` references instead of long inline strings to avoid quoting issues.
  - Capture stdout/stderr into log files or append them to a workflow log document.
- Prompting considerations:
  - Make budgets explicit per call (timebox, maximum commands, search limits) and encode them in the XML prompt.
  - Request structured output formats (for example, bullet summaries, patch snippets, and next-step recommendations).
  - Ensure all commands run from the project root unless a different directory is required.
- Safety:
  - Do not use destructive git subcommands; avoid undoing user changes.
  - Prefer reproducible commands (pin tool versions, use existing scripts) over ad hoc one-offs.

## Gemini CLI (Google)
- Overview:
  - Gemini CLI is an open-source AI agent that exposes Gemini models directly in the terminal and supports many coding workflows that other dev assistants provide.
  - Distributed as an npm package (`@google/gemini-cli`), configured via environment variables such as `GEMINI_API_KEY`.
- Typical usage patterns (from README/docs):
  - Interactive chat sessions for coding help, refactors, and explanations in the current directory.
  - File- and directory-aware operations where Gemini can read local context before generating changes.
  - Commands that propose changes and then apply them after user confirmation.
- Subagent orchestration pattern with Gemini CLI:
  - Store the subagent’s system prompt in an XML file and pass it through whichever configuration mechanism Gemini CLI exposes for system or role instructions.
  - Provide task-specific instructions via stdin or an input file, referencing the target files or directories.
  - Prefer non-interactive flows where the CLI prints a plan and proposed diff, and the orchestrator decides whether to apply or adjust, so behavior is easier to compare across harnesses.
  - Capture the CLI’s output and write it into the workflow log and agent artifact directory.
- Prompting considerations:
  - Ask Gemini to first output a plan (files to touch, operations, tests to run), then the proposed patch or commands.
  - Specify the project root, any excluded paths, and explicit safety requirements (no global installs, no secrets, confirmation before DB or deployment operations).
  - Request structured responses that separate analysis, decisions, and concrete output artifacts.
- Safety:
  - Configure the API key via environment or config file, not inline in prompts.
  - Validate generated commands before allowing them to run in automated scripts.

## Cross-Harness Best Practices for Subagent Calls
- Store prompt and payload in files and pass them via CLI file flags or redirection to avoid quoting drift and make runs reproducible.
- Enforce budgets per call: maximum number of shell commands, explicit time or token limits, and bounded search depth; encode these in the XML prompts.
- Keep context focused: pass only the minimum necessary excerpts (for example, key sections of README, design docs, or the relevant source files) to control latency and cost.
- Log each subagent invocation with enough structure to replay or compare behavior later (for example: `event | step | agent | command | result | decision | issue | lesson`).
- Apply the same safety rails across harnesses: avoid destructive git operations, require explicit confirmation for migrations or deployments, and use reproducible installation or build commands wherever possible.
