# Research: Calling Subagents via CLI in Target Harnesses

Scope: how to invoke subagents (LLM workers) from the main orchestrator when running inside common CLI-based agent harnesses. Harnesses considered: Claude Code (Anthropic), Codex CLI (this project harness), Gemini Code Assist (Google). Cursor/ArCursor excluded per latest guidance.

## Sources Consulted
- Anthropic Claude Code overview: https://docs.anthropic.com/claude/docs/claude-code (accessed via HTTP 200)
- Anthropic tool-calling docs: https://docs.anthropic.com/claude/docs/tools (HTTP 200)
- Google Gemini Code Assist product page: https://cloud.google.com/products/gemini-code-assist (HTTP 200; docs pages return 404)

## Claude Code (Anthropic)
- Product: Claude 3.5 Sonnet-powered dev environment with shell/editor integration (see Anthropic Claude Code docs).
- Shell execution: supports structured tool use; commands are echoed and executed in an attached terminal. Good for spawning subagents via CLI calls (e.g., `python script.py`, `curl` to APIs).
- Prompt handling: Claude Code favors concise, explicit instructions and benefits from a plan-before-run pattern. System prompts can be provided via files and referenced in the chat.
- Subagent call pattern (recommended):
  - Write subagent XML prompt to a temp file (e.g., `/tmp/subagent.xml`).
  - Invoke the subagent CLI with `--system` pointing to that file and pipe task payload via stdin.
  - Keep command count and token budgets explicit in the request so Claude respects limits.
- Safety: reiterate “no destructive git”, operate from repo root, and request confirmation for migrations/deployments.

## Codex CLI (current harness)
- Tooling: shell access with guidance to use `rg` for search and `apply_patch` for single-file edits. System prompt can be supplied via file content (e.g., `--system "$(cat prompt.xml)"`).
- Subagent call pattern:
  - Emit XML system prompt to temp file, task payload to another file.
  - Call the subagent process/model with explicit `--system` and `--input` arguments to avoid quoting issues.
  - Limit command count per step; summarize outputs with file:line references.
- Safety: avoid destructive git commands; do not undo user changes; prefer repo-root execution.

## Gemini Code Assist (Google)
- Public docs for CLI invocation are scarce; the product page (above) is reachable, but `cloud.google.com/gemini-code-assist/docs/*` currently returns 404. Historical/preview guidance suggests Cloud Shell/IDE integration with Gemini-backed code assist and limited CLI hooks.
- Likely CLI surface (based on Cloud code tooling patterns):
  - `gcloud` integration for chat/code assist is typically gated/preview; commands often look like `gcloud alpha genai code chat` or editor-integrated prompts rather than a raw `--system` flag.
  - Passing system prompts via files is recommended to preserve formatting; stdin for task payloads.
- Constraints: confirm product availability/permissions; avoid global installs; require confirmation for DB/deploy operations.

## Cross-Harness Best Practices for Subagent Calls
- Store prompt + payload in files to avoid shell quoting drift: `--system "$(cat prompt.xml)" --input "$(cat task.txt)"` or equivalent file flags if supported.
- Enforce budgets per call: max commands, token/time limits, search limits; surface them in the prompt.
- Context discipline: pass only relevant excerpts (<=2k tokens) to reduce latency and cost.
- Logging: capture `event | step | agent | command | result | decision | issue | lesson` for each subagent invocation.
- Safety rails: no destructive git; confirm before migrations/deploys/secrets access; prefer reproducible installs (`--frozen-lockfile`, `--no-optional`).

## Gaps / Follow-ups
- Gemini CLI specifics remain unclear; need updated Google documentation or access to a current preview (`gcloud` commands, flags for system prompts). If you have a tenant with Gemini Code Assist enabled, we can run `gcloud components list | rg gemini` and inspect available verbs.
- For Claude Code, obtain any harness-specific flags for providing system prompts via files once exposed in product documentation.
