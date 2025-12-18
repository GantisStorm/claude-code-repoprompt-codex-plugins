# Gemini Swarm

One-shot swarm commands with Gemini MCP planning using [jamubc/gemini-mcp-tool](https://github.com/jamubc/gemini-mcp-tool). No iterative loops or checkpoints - just parallel agent execution with Gemini's intelligent planning using `gemini-3-flash-preview`.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install gemini-swarm@claude-code-repoprompt-codex-plugins

# Install the Gemini MCP server
claude mcp add gemini-cli -- npx -y @anthropic/gemini-mcp-tool
```

## Commands

### /plan - Create Implementation Plan via Gemini

Spawns code-scout and doc-scout in parallel, then uses Gemini with the Architect system prompt to create an implementation plan.

```bash
/gemini-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js
/gemini-swarm:plan task:Fix login button not responding on mobile | research:React touch event handling
/gemini-swarm:plan task:Add OAuth2 login | research:Google OAuth2 Node.js
```

**Flow:** code-scout + doc-scout (parallel) -> Gemini planner (gemini-3-flash-preview) -> full plan output

**Output:** A complete implementation plan with per-file instructions, ready for `/code`.

### /code - Execute Implementation Plan

Takes the plan from `/plan` and spawns plan-coders in parallel to implement all files. Each coder receives its instructions directly.

```bash
/gemini-swarm:code plan:[paste plan from /plan output]
```

**Flow:** Parse plan -> spawn all plan-coders (parallel) -> collect results

**Output:** Status table showing which files were completed/blocked.

## Architecture Diagram

```
       /plan command                      /code command
             │                                  │
             ▼                                  ▼
     ┌───────────────┐                  ┌───────────────┐
     │  Parse Input  │                  │  Parse Plan   │
     │  task:        │                  │ files_to_edit │
     │  research:    │                  │files_to_create│
     └───────┬───────┘                  └───────┬───────┘
             │                                  │
             ▼                                  ▼
     ┌───────┴───────┐                  ┌───────┴───────┐
     │               │                  │               │
     ▼               ▼                  ▼               ▼
┌──────────┐  ┌──────────┐        ┌──────────┐  ┌──────────┐
│code-scout│  │doc-scout │        │plan-coder│  │plan-coder│
└────┬─────┘  └────┬─────┘        │  file1   │  │  file2   │
     │             │              │(receives │  │(receives │
     ▼             ▼              │ plan     │  │ plan     │
 CODE_CONTEXT  EXTERNAL_          │ directly)│  │ directly)│
     │         CONTEXT            └────┬─────┘  └────┬─────┘
     └──────┬──────┘                   │             │
            │                          ▼             ▼
            ▼                       COMPLETE      COMPLETE
     ┌───────────┐                     │             │
     │  planner  │                     └──────┬──────┘
     │  (Gemini  │                            │
     │ 3-flash)  │                            ▼
     └─────┬─────┘                    ┌───────────────┐
           │                          │ Results Table │
           ▼                          └───────────────┘
   ┌────────────────┐
   │   FULL PLAN    │
   │  + file lists  │
   └────────────────┘
```

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT |
| planner | Synthesize context, send to Gemini | mcp__gemini-cli__ask-gemini | Full plan + file lists |
| plan-coder | Implement single file | Read, Edit, Write, Glob, Grep, Bash | Status + verified |

## Plan Distribution

Plans are passed directly from `/plan` to `/code` - the orchestrator distributes per-file instructions to coders. This ensures each coder has exactly the instructions it needs.

**Note:** Gemini MCP is fully one-shot - it has NO conversation history or session continuation. Each `/plan` call is independent. The full plan must be copied and passed to `/code`.

The jamubc/gemini-mcp-tool is required for the planner to communicate with Gemini, but coders don't need MCP access.

## Tips

**Getting good results with /plan:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- Include relevant research: "JWT best practices Node.js" not just "JWT"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari"

**Using /code:**
- Copy the Implementation Plan section from `/plan` output
- If some files are BLOCKED, fix issues and re-run `/plan` with a modified task
- Each coder receives its instructions directly - no MCP needed

**When things go wrong:**
- BLOCKED status includes error details - read them
- Check if Gemini MCP server is installed: `claude mcp list` should show `gemini-cli`
- Try regenerating the plan with more specific task/research

## Comparison with Other Plugins

| Feature | gemini-swarm | codex-swarm | gemini-pair-pipeline |
|---------|--------------|-------------|----------------------|
| Execution | One-shot | One-shot | Iterative with checkpoints |
| Planning | Gemini MCP (3-flash-preview) | Codex MCP (gpt-5.2) | Gemini MCP (3-flash-preview) |
| User control | Review plan, then execute | Review plan, then execute | Checkpoints during discovery |
| Commands | /plan + /code (separate) | /plan + /code (separate) | /orchestrate (all-in-one) |
| Use case | Well-defined tasks with Gemini | Well-defined tasks with Codex | Exploratory tasks |

Use **gemini-swarm** when you know what you want and want Gemini's architectural planning.

Use **codex-swarm** when you prefer OpenAI's Codex with high reasoning effort.

Use **gemini-pair-pipeline** when you need iterative discovery with user checkpoints.

## Requirements

- **Gemini MCP** - [jamubc/gemini-mcp-tool](https://github.com/jamubc/gemini-mcp-tool) - Install with `claude mcp add gemini-cli -- npx -y @anthropic/gemini-mcp-tool`
- **Claude Code** - Orchestration and execution
