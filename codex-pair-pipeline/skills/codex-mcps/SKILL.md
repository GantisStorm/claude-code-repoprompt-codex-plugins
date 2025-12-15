---
name: codex-mcps
description: Codex MCP tool reference. Supplemental parameter docs for codex and codex-reply tools.
allowed-tools: mcp__codex__codex, mcp__codex__codex-reply
---

# Codex MCP Tools Reference

Quick reference for Codex MCP tools. Run Codex as an MCP server with `codex mcp-server`.

---

## `mcp__codex__codex`

Run a Codex session. Accepts configuration parameters matching the Codex Config struct.

**Tips:**
- Use `developer-instructions` to set the Architect system prompt
- Use `model: "gpt-5.2"` with `config: {"model_reasoning_effort": "high"}` for architectural planning (recommended for complex tasks)
- Use `approval-policy: "never"` for automated pipelines
- Use `sandbox: "workspace-write"` to allow file modifications
- Codex sessions can take 30s-5min depending on task complexity

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | The initial user prompt to start the Codex conversation |
| `approval-policy` | No | `"untrusted"`, `"on-failure"`, `"on-request"`, `"never"` |
| `base-instructions` | No | Instructions to use instead of default ones |
| `compact-prompt` | No | Prompt used when compacting the conversation |
| `config` | No | Individual config settings that override config.toml |
| `cwd` | No | Working directory for the session |
| `developer-instructions` | No | Developer instructions injected as developer role message |
| `model` | No | Override for model name (e.g., "o3", "o4-mini") |
| `profile` | No | Configuration profile from config.toml |
| `sandbox` | No | `"read-only"`, `"workspace-write"`, `"danger-full-access"` |

**Returns:** Conversation events and final result. Includes `conversationId` for follow-up with `codex-reply`.

---

## `mcp__codex__codex-reply`

Continue a Codex conversation by providing the conversation id and prompt.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `conversationId` | Yes | The conversation id from a previous `codex` call |
| `prompt` | Yes | The next user prompt to continue the conversation |

**Returns:** Conversation events and final result for the continued session.

---

## Architect System Prompt

Use this as `developer-instructions` when calling `mcp__codex__codex` for architectural planning:

```
You are a senior software architect specializing in code design and implementation planning. Your role is to:

1. Analyze the requested changes and break them down into clear, actionable steps
2. Create a detailed implementation plan that includes:
   - Files that need to be modified
   - Specific code sections requiring changes
   - New functions, methods, or classes to be added
   - Dependencies or imports to be updated
   - Data structure modifications
   - Interface changes
   - Configuration updates

For each change:
- Describe the exact location in the code where changes are needed
- Explain the logic and reasoning behind each modification
- Provide example signatures, parameters, and return types
- Note any potential side effects or impacts on other parts of the codebase
- Highlight critical architectural decisions that need to be made

You may include short code snippets to illustrate specific patterns, signatures, or structures, but do not implement the full solution.

Focus solely on the technical implementation plan - exclude testing, validation, and deployment considerations unless they directly impact the architecture.
```

---

## Timeout Considerations

Codex operations can take significant time. Recommended timeouts:
- Simple tasks: 60-120 seconds
- Complex tasks: 300-600 seconds (5-10 minutes)

The MCP Inspector recommends setting Request and Total timeouts to 600000ms (10 minutes) under Configuration.

---

## Prompt Structure for Context

Since Codex doesn't handle context as well as some other tools, include CODE_CONTEXT explicitly:

```
<CODE_CONTEXT>
[Full codebase context from code-scout]
</CODE_CONTEXT>

<EXTERNAL_CONTEXT>
[External documentation from doc-scout]
</EXTERNAL_CONTEXT>

<Q_AND_A>
[User answers to clarification questions]
</Q_AND_A>

<USER_INSTRUCTIONS>
[The architectural narrative describing what to build]
</USER_INSTRUCTIONS>
```

---

## Differences from RepoPrompt

| Feature | Codex MCP | RepoPrompt MCP |
|---------|-----------|----------------|
| Chat history fetch | Not available | `chats` tool with `action: "log"` |
| Context handling | Requires explicit CODE_CONTEXT | Built-in context_builder |
| File selection | Manual in prompt | Automatic with manage_selection |
| Plan persistence | Orchestrator must store | Persisted in chat history |

Because Codex doesn't have a chat history fetch API, the orchestrator must:
1. Store the full plan from planner response
2. Pass the plan directly to plan-coders in their prompts
