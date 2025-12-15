# Codex Pair Pipeline

Coordinates specialized agents to implement complex, multi-file coding tasks with Codex MCP. Uses **iterative discovery** where users build context incrementally, then Codex creates detailed architectural plans using the Architect system prompt.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install codex-pair-pipeline@claude-code-repoprompt-codex-plugins
```

## Commands

### command:start - Discovery Loop

Iterative discovery with checkpoints. Best for complex features.

```bash
/codex-pair-pipeline:orchestrate command:start | task:Add user authentication with JWT tokens
/codex-pair-pipeline:orchestrate command:start | task:Fix login button not responding on mobile
/codex-pair-pipeline:orchestrate command:start | task:Add OAuth2 login | research:Google OAuth2 best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> Codex planning -> execution

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> Codex planning -> execution

### command:start-resume - Continue Previous Plan

Discovery loop, then continue in the same Codex conversation. Preserves conversation context.

```bash
/codex-pair-pipeline:orchestrate command:start-resume | task:Add password reset flow
/codex-pair-pipeline:orchestrate command:start-resume | task:Add rate limiting to the new endpoints
/codex-pair-pipeline:orchestrate command:start-resume | task:Add email verification | research:SendGrid API best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> Codex planning -> execution (continues existing Codex conversation)

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> Codex planning -> execution

### command:fetch - Execute Existing Plan

Execute an existing Codex plan by conversation ID.

```bash
/codex-pair-pipeline:orchestrate command:fetch | task:conv-abc123-xyz
```

**Flow:** planner-fetch -> execution

## How It Works

The orchestrator spawns specialized agents via the `Task` tool:

1. **Discovery** - code-scout gathers CODE_CONTEXT; doc-scout adds EXTERNAL_CONTEXT (spawned in parallel when `research:` is provided upfront, otherwise optional at checkpoint)
2. **Checkpoints** - User answers clarifying questions, decides when context is complete
3. **Planning** - Codex (using `gpt-5.2` model with high reasoning effort) receives Architect system prompt + CODE_CONTEXT + architectural narrative and creates detailed plan
4. **Execution** - Coders implement in parallel, verify with code-quality skill

| Command | Discovery | Planning |
|---------|-----------|----------|
| `command:start` | Checkpoints + optional research | Codex (planner-start) |
| `command:start-resume` | Checkpoints + optional research | Codex (planner-start-resume) |
| `command:fetch` | None | Fetches existing plan |

## Architecture Diagram

```
                                         ORCHESTRATOR
                                              |
                +-----------------------------+-----------------------------+
                |                                                           |
                v                                                           v
      +==================+                                        +==================+
      | command:start    |                                        | command:fetch    |
      | command:start-   |                                        |                  |
      |         resume   |                                        |                  |
      +========+=========+                                        +========+=========+
               |                                                           |
               | task:                                                     | conversation_id:
               v                                                           |
+================================+                                         |
|  ITERATIVE DISCOVERY LOOP      |                                         |
|  (Human-in-the-loop control)   |                                         |
|                                |                                         |
|  +-------------------+         |                                         |
|  | code-scout        |         |                                         |
|  | (mode: info/dir)  |         |                                         |
|  +---------+---------+         |                                         |
|            |                   |                                         |
|            | CODE_CONTEXT      |                                         |
|            v                   |                                         |
|  +-------------------+         |                                         |
|  | CHECKPOINT        |         |                                         |
|  | (AskUserQuestion) |         |                                         |
|  | 1. Clarifying Qs  |         |                                         |
|  | 2. "Add research" |         |                                         |
|  |  or "Complete"    |         |                                         |
|  +---------+---------+         |                                         |
|            |                   |                                         |
|       +----+----+              |                                         |
|       |         |              |                                         |
|       v         |              |                                         |
|  +---------+    |              |                                         |
|  |doc-scout|    |              |                                         |
|  +----+----+    |              |                                         |
|       |         |              |                                         |
|       | EXTERNAL_CONTEXT       |                                         |
|       v         |              |                                         |
|  +-------------------+         |                                         |
|  | CHECKPOINT        |         |                                         |
|  | (AskUserQuestion) |         |                                         |
|  | 1. Clarifying Qs  |         |                                         |
|  | 2. "Add more" or  |         |                                         |
|  |    "Complete"     |         |                                         |
|  +----+----+---------+         |                                         |
|       |    |                   |                                         |
|       |    +---> (loop back)   |                                         |
|       |                        |                                         |
|       | context_package        |                                         |
+========+=======================+                                         |
         |                                                                 |
         v                                                                 v
+------------------+                                          +------------------+
| planner-start    |                                          | planner-fetch    |
| or planner-start-|                                          |                  |
|          resume  |                                          | conversation_id: |
|                  |                                          |                  |
| instructions:    |                                          +--------+---------+
| context_package  |                                                   |
+--------+---------+                                                   |
         |                                                             |
         | conversation_id +                                           | conversation_id +
         | file_lists                                                  | file_lists
         v                                                             v
+------------------+                                          +------------------+
| plan-coder       |                                          | plan-coder       |
| (1/file)         |                                          | (1/file)         |
|                  |                                          |                  |
| Input:           |                                          | Input:           |
|  conversation_id |                                          |  conversation_id |
|  target_file     |                                          |  target_file     |
|  action          |                                          |  action          |
|                  |                                          |                  |
| Fetches plan     |                                          | Fetches plan     |
| from Codex       |                                          | from Codex       |
+--------+---------+                                          +--------+---------+
         |                                                             |
         | status:                                                     | status:
         | COMPLETE/BLOCKED                                            | COMPLETE/BLOCKED
         |                                                             |
         +----------------------------+--------------------------------+
                                      |
                                      v
                              +------------------+
                              |   ORCHESTRATOR   |
                              |  Phase 3 Review  |
                              |                  |
                              | Collects results |
                              | Reports to user  |
                              +------------------+
```

**Key flows:**
- `command:start` -> discovery -> planner-start -> plan-coder
- `command:start-resume` -> discovery -> planner-start-resume -> plan-coder
- `command:fetch` -> planner-fetch -> plan-coder

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT + clarification |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT + clarification |
| planner-start | Synthesize prompt, create plan via Codex | mcp__codex__codex | conversation_id + file lists |
| planner-start-resume | Synthesize prompt for new task in existing conversation | mcp__codex__codex-reply | conversation_id + file lists |
| planner-fetch | Fetch existing plan (NO synthesis) | mcp__codex__codex-reply | conversation_id + file lists |
| plan-coder | Implement single file (Codex mode) | Read, Edit, Write, Glob, Grep, Bash, mcp__codex__codex-reply | status + verified |

## Architect System Prompt

The planner agents send the Architect system prompt to Codex as developer-instructions:

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

## Context Handling

Since Codex doesn't handle context as well as RepoPrompt, the planner agents include the full CODE_CONTEXT in the prompt sent to Codex, ensuring the architect has all the necessary codebase information to create accurate plans.

## Tips

**Choosing the right command:**
- `command:start` - Explore unfamiliar code, want checkpoints
- `command:start-resume` - Continue in same Codex conversation
- `command:fetch` - Re-execute existing plan

**Getting good results:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari" not "Login broken"
- Use `research:` for external APIs

**When things go wrong:**
- BLOCKED status includes error details - read them
- Re-run with `command:start-resume` after fixing blockers
- Incomplete context? Add research at checkpoints

## See Also

For RepoPrompt-based planning, see **repoprompt-pair-pipeline** which uses RepoPrompt's context_builder for planning.

## Requirements

- **Codex MCP** - Required for all planning modes (run `codex mcp-server`)
- **Claude Code** - Orchestration, discovery, and execution
