# Codex Pair Pipeline

Coordinates specialized agents to implement complex, multi-file coding tasks with Codex MCP using [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server). Uses **iterative discovery** where users build context incrementally, then Codex creates detailed architectural plans using the Architect system prompt.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install codex-pair-pipeline@claude-code-repoprompt-codex-plugins

# Install the Codex MCP server
claude mcp add codex-cli -- npx -y codex-mcp-server
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

Discovery loop, then continue in the same Codex session. Preserves session context.

```bash
/codex-pair-pipeline:orchestrate command:start-resume | task:Add password reset flow
/codex-pair-pipeline:orchestrate command:start-resume | task:Add rate limiting to the new endpoints
/codex-pair-pipeline:orchestrate command:start-resume | task:Add email verification | research:SendGrid API best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> Codex planning -> execution (continues existing Codex session)

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> Codex planning -> execution

### command:fetch - Execute Existing Plan

Execute an existing Codex plan by session ID.

```bash
/codex-pair-pipeline:orchestrate command:fetch | task:session-abc123-xyz
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
               | task:                                                     | session_id:
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
|          resume  |                                          | session_id:      |
|                  |                                          |                  |
| instructions:    |                                          +--------+---------+
| context_package  |                                                   |
+--------+---------+                                                   |
         |                                                             |
         | session_id +                                                | session_id +
         | file_lists                                                  | file_lists
         v                                                             v
+------------------+                                          +------------------+
| plan-coder       |                                          | plan-coder       |
| (1/file)         |                                          | (1/file)         |
|                  |                                          |                  |
| Input:           |                                          | Input:           |
|  session_id      |                                          |  session_id      |
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
| planner-start | Synthesize prompt, create plan via Codex | mcp__codex-cli__codex | session_id + file lists |
| planner-start-resume | Synthesize prompt for new task in existing session | mcp__codex-cli__codex | session_id + file lists |
| planner-fetch | Fetch existing plan (NO synthesis) | mcp__codex-cli__codex | session_id + file lists |
| plan-coder | Implement single file (Codex mode) | Read, Edit, Write, Glob, Grep, Bash, mcp__codex-cli__codex | status + verified |

## Session Continuation

The tuannvm/codex-mcp-server provides proper session support (unlike the official server which has [Issue #3712](https://github.com/openai/codex/issues/3712)):

- **sessionId** returned in responses for follow-up calls
- **24-hour session persistence**
- Use `listSessions` tool to see active sessions

## Tips

**Choosing the right command:**
- `command:start` - Explore unfamiliar code, want checkpoints
- `command:start-resume` - Continue in same Codex session
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

- **Codex MCP** - [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) - Install with `claude mcp add codex-cli -- npx -y codex-mcp-server`
- **Claude Code** - Orchestration, discovery, and execution
