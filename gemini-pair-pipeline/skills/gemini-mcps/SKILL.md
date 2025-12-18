---
name: gemini-mcps
description: Gemini MCP tool reference. Supplemental parameter docs for ask-gemini tool (jamubc/gemini-mcp-tool).
allowed-tools: mcp__gemini-cli__ask-gemini
---

# Gemini MCP Tools Reference (jamubc/gemini-mcp-tool)

Quick reference for Gemini MCP tools. Install with: `claude mcp add gemini-cli -- npx -y @anthropic/gemini-mcp-tool`

---

## `mcp__gemini-cli__ask-gemini`

Run a Gemini query with optional model selection and sandbox mode.

**Tips:**
- Use `model: "gemini-3-flash-preview"` for architectural planning
- Gemini is fully one-shot - no session history or conversation continuation
- All context must be included in each prompt

| Parameter | Required | Description |
|-----------|----------|-------------|
| `prompt` | Yes | The full prompt including system instructions and all context |
| `model` | No | Model to use. Use `gemini-3-flash-preview` |

**Returns:** Response text with architectural plan.

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

## Key Differences from Codex MCP

| Feature | Gemini MCP | Codex MCP |
|---------|------------|-----------|
| Session continuation | None - fully one-shot | Via sessionId |
| Model | gemini-3-flash-preview | gpt-5.1-codex, gpt-5.2 |
| Context management | All context in prompt | Session maintains history |

**Important:** Gemini MCP has no conversation history. Every call is independent. The orchestrator must manage all context and pass it explicitly to planners:
- For `command:start`: Pass CODE_CONTEXT, EXTERNAL_CONTEXT, Q&A
- For `command:continue`: Also include previous plan summaries from orchestrator memory

The planner must return the full plan for each request.
