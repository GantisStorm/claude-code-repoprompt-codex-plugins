---
name: repoprompt-mcps
description: RepoPrompt MCP tool reference. Supplemental parameter docs for context_builder, chat_send, chats, and workspace management tools.
allowed-tools: mcp__RepoPrompt__context_builder, mcp__RepoPrompt__chat_send, mcp__RepoPrompt__chats, mcp__RepoPrompt__manage_workspaces, mcp__RepoPrompt__manage_selection, mcp__RepoPrompt__workspace_context, mcp__RepoPrompt__get_file_tree, mcp__RepoPrompt__get_code_structure, mcp__RepoPrompt__file_search, mcp__RepoPrompt__read_file
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

---

## `mcp__RepoPrompt__manage_workspaces`

Manage workspaces across RepoPrompt windows. Use `select_tab` to bind your MCP connection to a specific compose tab for consistent context.

**Key actions:**
- `list_tabs` - List compose tabs for the active workspace
- `select_tab` - Bind this MCP connection to a specific tab (recommended for stable context)

| Parameter | Required | Description |
|-----------|----------|-------------|
| `action` | Yes | `"list"` \| `"switch"` \| `"create"` \| `"delete"` \| `"add_folder"` \| `"remove_folder"` \| `"list_tabs"` \| `"select_tab"` |
| `tab` | For select_tab | Tab UUID or name |
| `workspace` | For switch/delete/folder ops | Workspace UUID or name |
| `focus` | No | For select_tab: if true, switches UI to show selected tab (default: false) |

**Tab binding:**
- Without binding, tools use the user's currently active tab (can be inconsistent)
- Use `select_tab` to pin your connection to a specific tab for predictable operations

---

## `mcp__RepoPrompt__manage_selection`

Manage the current file selection used by all tools.

**Operations:**
- `get` - View current selection
- `add` - Add files/directories to selection
- `remove` - Remove files from selection
- `set` - Replace entire selection
- `clear` - Clear selection
- `promote` - Upgrade codemap to full file
- `demote` - Downgrade full file to codemap

**Modes:**
- `full` - Complete file content (default)
- `slices` - Specific line ranges only
- `codemap_only` - Signatures only (efficient for reference files)

| Parameter | Required | Description |
|-----------|----------|-------------|
| `op` | Yes | `"get"` \| `"add"` \| `"remove"` \| `"set"` \| `"clear"` \| `"preview"` \| `"promote"` \| `"demote"` |
| `paths` | For add/remove/set | File or folder paths |
| `mode` | No | `"full"` \| `"slices"` \| `"codemap_only"` |
| `view` | No | `"summary"` \| `"files"` \| `"content"` |
| `slices` | For slice mode | Array of `{path, ranges: [{start_line, end_line, description}]}` |

**Examples:**
```json
// Add full file
{"op": "add", "paths": ["src/auth/UserAuth.ts"]}

// Get current selection with file details
{"op": "get", "view": "files"}

// Promote codemap to full file
{"op": "promote", "paths": ["src/models/User.ts"]}

// Add slices for large file
{"op": "set", "mode": "slices", "slices": [
  {"path": "src/auth/Auth.ts", "ranges": [
    {"start_line": 45, "end_line": 120, "description": "UserAuth class - login/logout"}
  ]}
]}
```

---

## `mcp__RepoPrompt__workspace_context`

Snapshot of workspace state: prompt, selection, code structure (codemaps), tokens.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `include` | No | Array: `["prompt", "selection", "code", "files", "tree", "tokens"]` (defaults to prompt, selection, code, tokens) |
| `path_display` | No | `"relative"` \| `"full"` |

**Returns:** Current token count, selected files, prompt content, and code structure.

---

## `mcp__RepoPrompt__get_file_tree`

ASCII directory tree of the project.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `type` | Yes | `"files"` \| `"roots"` |
| `mode` | No | For files: `"auto"` \| `"full"` \| `"folders"` \| `"selected"` |
| `path` | No | Starting folder for subtree |
| `max_depth` | No | Maximum depth (root = 0) |

**Examples:**
```json
// Auto tree (fits ~10k tokens)
{"type": "files", "mode": "auto"}

// Drill into specific directory
{"type": "files", "mode": "auto", "path": "src/components", "max_depth": 2}
```

---

## `mcp__RepoPrompt__get_code_structure`

Return code structure (codemaps) for files and directories.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `scope` | No | `"selected"` (current selection) or `"paths"` (explicit paths, default) |
| `paths` | For paths scope | Array of file/directory paths |
| `max_results` | No | Maximum codemaps to return (default: 25) |

---

## `mcp__RepoPrompt__file_search`

Search by file path and/or content.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `pattern` | Yes | Search pattern (regex by default) |
| `mode` | No | `"auto"` \| `"path"` \| `"content"` \| `"both"` |
| `regex` | No | Use regex matching (default: true) |
| `max_results` | No | Maximum results (default: 50) |
| `context_lines` | No | Lines of context around matches |

**Examples:**
```json
// Find files containing UserAuth
{"pattern": "UserAuth", "mode": "content"}

// Find Swift files
{"pattern": "*.swift", "mode": "path", "regex": false}
```

---

## `mcp__RepoPrompt__read_file`

Read file contents with optional line range.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `path` | Yes | File path |
| `start_line` | No | Line to start from (1-based), negative for tail |
| `limit` | No | Number of lines to read |

**Examples:**
```json
// Read entire file
{"path": "src/auth/UserAuth.ts"}

// Read lines 45-120
{"path": "src/auth/UserAuth.ts", "start_line": 45, "limit": 76}
```
