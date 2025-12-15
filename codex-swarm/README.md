# Codex Swarm

One-shot swarm commands with Codex MCP planning. No iterative loops or checkpoints - just parallel agent execution with Codex's intelligent planning using `gpt-5.2` with high reasoning effort.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install codex-swarm@claude-code-repoprompt-codex-plugins
```

## Commands

### /plan - Create Implementation Plan via Codex

Spawns code-scout and doc-scout in parallel, then uses Codex with the Architect system prompt to create an implementation plan.

```bash
/codex-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js
/codex-swarm:plan task:Fix login button not responding on mobile | research:React touch event handling
/codex-swarm:plan task:Add OAuth2 login | research:Google OAuth2 Node.js
```

**Flow:** code-scout + doc-scout (parallel) -> Codex planner (gpt-5.2) -> conversation_id output

**Output:** A conversation_id referencing the Codex plan, ready for `/code`.

### /code - Execute Codex Plan

Takes a conversation_id and spawns plan-coders in parallel to implement all files. Coders fetch their instructions from Codex.

```bash
/codex-swarm:code conversation_id:[conversation_id from /plan output]
```

**Flow:** Fetch plan from Codex -> spawn all plan-coders (parallel) -> collect results

**Output:** Status table showing which files were completed/blocked.

## Architecture Diagram

```
/plan command                          /code command
     |                                      |
     v                                      v
+==============+                    +================+
| Parse Input  |                    | Fetch Plan     |
| task:        |                    | from Codex     |
| research:    |                    | (conv_id)      |
+======+======+                    +========+=======+
       |                                    |
       v                                    v
+------+------+                    +--------+--------+
|             |                    | Parse file lists|
v             v                    +--------+--------+
+----------+ +----------+                   |
|code-scout| |doc-scout |                   v
+----+-----+ +----+-----+          +--------+--------+
     |            |                |                 |
     |            |                v                 v
     v            v         +-----------+    +-----------+
  CODE_CONTEXT  EXTERNAL_   |plan-coder |    |plan-coder |
     |          CONTEXT     |  file1    |    |  file2    |
     +-----+------+         | (fetches  |    | (fetches  |
           |                |  from     |    |  from     |
           v                |  Codex)   |    |  Codex)   |
     +----------+           +-----+-----+    +-----+-----+
     | planner  |                 |                |
     | (uses    |                 v                v
     | Codex    |             COMPLETE          COMPLETE
     | gpt-5.2) |                 |                |
     |          |                 +-------+--------+
     +----+-----+                         |
          |                               v
          v                       +================+
  +=================+             | Results Table  |
  | CONVERSATION_ID |             +================+
  | + file lists    |
  +=================+
```

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT |
| planner | Synthesize context, send to Codex | mcp__codex__codex | conversation_id + file lists |
| plan-coder | Implement single file (fetches from Codex) | Read, Edit, Write, Glob, Grep, Bash, mcp__codex__codex-reply | Status + verified |

## Architect System Prompt

The planner sends the Architect system prompt to Codex as `developer-instructions`:

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

Since Codex doesn't handle context as well as RepoPrompt, the planner includes the full CODE_CONTEXT in the prompt sent to Codex, ensuring the architect has all the necessary codebase information to create accurate plans.

## Tips

**Getting good results with /plan:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- Include relevant research: "JWT best practices Node.js" not just "JWT"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari"

**Using /code:**
- The plan is stored in Codex - just pass the conversation_id
- If some files are BLOCKED, fix issues and re-run `/code` with the same conversation_id
- Each coder fetches its instructions from Codex independently

**When things go wrong:**
- BLOCKED status includes error details - read them
- Check if Codex MCP server is running (`codex mcp-server`)
- Try regenerating the plan with more specific task/research

## Comparison with Other Plugins

| Feature | codex-swarm | repoprompt-swarm | codex-pair-pipeline |
|---------|-------------|------------------|---------------------|
| Execution | One-shot | One-shot | Iterative with checkpoints |
| Planning | Codex MCP (gpt-5.2) | RepoPrompt MCP | Codex MCP (gpt-5.2) |
| User control | Review plan, then execute | Review plan, then execute | Checkpoints during discovery |
| Commands | /plan + /code (separate) | /plan + /code (separate) | /orchestrate (all-in-one) |
| Use case | Well-defined tasks with Codex | Well-defined tasks with RepoPrompt | Exploratory tasks |

Use **codex-swarm** when you know what you want and want Codex's gpt-5.2 architectural planning with high reasoning effort.

Use **repoprompt-swarm** when you prefer RepoPrompt's context handling.

Use **codex-pair-pipeline** when you need iterative discovery with user checkpoints.

## Requirements

- **Codex MCP** - Required for planning and plan retrieval (run `codex mcp-server`)
- **Claude Code** - Orchestration and execution
