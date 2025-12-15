---
name: codex-mcps
description: Codex MCP tool reference. Supplemental parameter docs for codex tool (tuannvm/codex-mcp-server).
allowed-tools: mcp__codex-cli__codex, mcp__codex-cli__listSessions
---

# Codex MCP Tools Reference (tuannvm/codex-mcp-server)

Quick reference for Codex MCP tools. Install with: `claude mcp add codex-cli -- npx -y codex-mcp-server`

---

## `mcp__codex-cli__codex`

Run a Codex session with optional session support for multi-turn conversations.

**Tips:**
- Use `sessionId` to continue an existing conversation
- Use `model: "gpt-5.2"` with `reasoningEffort: "high"` for architectural planning
- Use `resetSession: true` to clear session history before processing
- Sessions persist for 24 hours

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | The coding question or request |
| `sessionId` | No | Session ID for conversation continuation |
| `resetSession` | No | Clear session history before processing |
| `model` | No | Model to use (defaults to gpt-5.1-codex) |
| `reasoningEffort` | No | Reasoning effort: "minimal", "low", "medium", "high" |

**Returns:** Response text and `sessionId` for follow-up calls.

---

## `mcp__codex-cli__listSessions`

List active conversation sessions with metadata.

| Parameter | Required | Description |
|-----------|----------|-------------|
| (none) | - | No parameters required |

**Returns:** List of sessions with creation time, last access, and turn count.

---

## Prompt Structure for Context

Include the Architect system prompt as a preamble, followed by CODE_CONTEXT:

```
<SYSTEM>
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
</SYSTEM>

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

## Session Continuation

To continue a conversation, pass the `sessionId` from a previous response:

```
mcp__codex-cli__codex with:
- sessionId: [from previous response]
- prompt: "Please provide implementation details for [specific file]"
```

The server uses native `codex resume` functionality for conversation continuity.

---

## Timeout Considerations

Codex operations can take significant time. Recommended timeouts:
- Simple tasks: 60-120 seconds
- Complex tasks: 300-600 seconds (5-10 minutes)

---

## Key Differences from Official codex mcp-server

| Feature | tuannvm/codex-mcp-server | Official codex mcp-server |
|---------|--------------------------|---------------------------|
| Session ID returned | Yes (in response) | No (broken - Issue #3712) |
| Session continuation | Works via sessionId | Broken |
| listSessions tool | Yes | No |
| reasoningEffort param | Yes (direct) | Via config object |
| Model selection | Direct parameter | Direct parameter |

---

## Differences from RepoPrompt

| Feature | Codex MCP | RepoPrompt MCP |
|---------|-----------|----------------|
| Chat history fetch | Via listSessions | `chats` tool with `action: "log"` |
| Context handling | Requires explicit CODE_CONTEXT | Built-in context_builder |
| File selection | Manual in prompt | Automatic with manage_selection |
| Plan persistence | Via sessionId continuation | Persisted in chat history |

Because Codex supports session continuation via sessionId, the orchestrator can:
1. Store the sessionId from planner response
2. Pass sessionId to plan-coders for fetching implementation details
