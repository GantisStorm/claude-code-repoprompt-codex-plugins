---
name: repoprompt-mcps
description: RepoPrompt MCP tool reference. Supplemental parameter docs for context_builder, chat_send, and chats tools.
allowed-tools: mcp__RepoPrompt__context_builder, mcp__RepoPrompt__chat_send, mcp__RepoPrompt__chats
---

# RepoPrompt MCP Tools Reference

Quick reference for RepoPrompt MCP tools.

---

## `mcp__RepoPrompt__context_builder`

Start a task by intelligently exploring the codebase and building optimal file context. Ideal as the first step for complex tasks: the agent maps out relevant code, selects files within a token budget, and can generate an implementation plan in one shot.

**Tips:**
- Use `response_type="plan"` to get an implementation plan alongside the context
- Continue the conversation via `chat_send` with the returned `chat_id`
- Add files to the selection with `manage_selection` as new areas become relevant
- Run `context_builder` again anytime to explore a different area of the codebase

**Note:** Thorough exploration takes 30s-5min depending on codebase size and task complexity.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `instructions` | Yes | Task description for the agent (what context to build) |
| `response_type` | No | `"plan"` for implementation plan, `"question"` to ask about codebase, omit for context-only |

**Returns:** File selection, prompt, status. With `response_type="plan"` or `"question"`, also returns response and `chat_id` for follow-up.

---

## `mcp__RepoPrompt__chat_send`

Start a new chat or continue an existing conversation.

**Modes:**
- `plan` - Architectural planning without immediate edits
- `edit` - Generate and apply code changes
- `chat` - General conversation

**Session management:**
- `new_chat: true` to start fresh; else continues most recent (or pass `chat_id`)
- `chat_name` is optional but recommended - short, descriptive (e.g., "Fix login crash - auth flow")
- `model` - preset id/name; call `list_models` first

**Limitations:**
- No commands/tests; only sees selected files + conversation history
- Chat does not track diff history; it sees current file state only

| Parameter | Required | Description |
|-----------|----------|-------------|
| `new_chat` | Yes | `true` to start new session, `false` to continue |
| `message` | Yes* | Your message to send (*ignored if `use_tab_prompt` is true) |
| `mode` | No | `"chat"` \| `"plan"` \| `"edit"` |
| `chat_id` | No | Chat ID to continue a specific chat |
| `chat_name` | No | Set or update the chat session name |
| `model` | No | Model preset ID or name |
| `selected_paths` | No | File paths for context (overrides current selection) |
| `include_diffs` | No | Include edit diffs in reply (default: true) |
| `use_tab_prompt` | No | When true, use active compose tab's prompt instead of message |

**Returns:** Chat response, updated plan (if mode="plan"), or applied edits (if mode="edit")

---

## `mcp__RepoPrompt__chats`

List chats or view a chat's history.

**Actions:**
- `list` - Recent chats (ID, name, selected files, last activity) - use the ID with `chat_send`
- `log` - Full conversation history (optionally include diffs)

**Notes:**
- Each chat maintains its own selection and context; continuing restores state
- `chat_send` without `chat_id` resumes the most recent by default
- Rename a session anytime by passing `chat_name` in `chat_send`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `action` | Yes | `"list"` or `"log"` |
| `chat_id` | For log | Chat ID to retrieve (alias: `id`) |
| `limit` | No | Max items - list: sessions (default 10), log: messages (default 3) |
| `diffs` | No | Include diff summaries (alias: `include_diffs`) |

**Returns:** Messages array in chronological order

**Parsing rules (for log action):**
- `role: "user"` = task context (ignore)
- `role: "assistant"` = architectural plan (parse this)
- Always use the **last** assistant message
