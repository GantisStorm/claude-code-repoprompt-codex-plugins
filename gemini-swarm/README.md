# Gemini Swarm

One-shot swarm commands with Gemini CLI planning. No iterative loops or checkpoints - just parallel agent execution with Gemini's intelligent planning using `gemini-3-flash-preview`.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install gemini-swarm@claude-code-repoprompt-codex-plugins
```

## Requirements

- **Gemini CLI** - Install with `npm install -g @google/gemini-cli` or `brew install gemini-cli` (macOS)
- **Claude Code** - Orchestration and execution

## Commands

### /plan - Create Implementation Plan via Gemini

Spawns code-scout and doc-scout as background tasks (parallel), then uses Gemini CLI with the Architect system prompt to create an implementation plan. Uses `TaskOutput` to retrieve results.

```bash
/gemini-swarm:plan task:Add user authentication with JWT tokens | research:JWT best practices Node.js
/gemini-swarm:plan task:Fix login button not responding on mobile | research:React touch event handling
/gemini-swarm:plan task:Add OAuth2 login | research:Google OAuth2 Node.js
```

**Flow:** code-scout + doc-scout (parallel) -> Gemini planner (gemini-3-flash-preview) -> full plan output

**Output:** A complete implementation plan with per-file instructions, ready for `/code`.

### /code - Execute Implementation Plan

Takes the plan from `/plan` and spawns plan-coders as background tasks (parallel). Uses `TaskOutput` to collect results. Each coder receives its instructions directly.

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
     │  BACKGROUND   │                  │  BACKGROUND   │
     │    SPAWN      │                  │    SPAWN      │
     ▼               ▼                  ▼               ▼
┌──────────┐  ┌──────────┐        ┌──────────┐  ┌──────────┐
│code-scout│  │doc-scout │        │plan-coder│  │plan-coder│
│run_in_   │  │run_in_   │        │  file1   │  │  file2   │
│background│  │background│        │run_in_   │  │run_in_   │
│  :true   │  │  :true   │        │background│  │background│
└────┬─────┘  └────┬─────┘        └────┬─────┘  └────┬─────┘
     │             │                   │             │
     ▼             ▼                   ▼             ▼
┌─────────────────────┐          ┌─────────────────────┐
│    TaskOutput       │          │    TaskOutput       │
│  (collect results)  │          │  (collect results)  │
└──────────┬──────────┘          └──────────┬──────────┘
           │                                │
 CODE_CONTEXT + EXTERNAL_CONTEXT    COMPLETE + COMPLETE
           │                                │
           ▼                                ▼
     ┌───────────┐                  ┌───────────────┐
     │  planner  │                  │ Results Table │
     │  (Gemini  │                  └───────────────┘
     │ 3-flash)  │
     └─────┬─────┘
           │
           ▼
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
| planner | Synthesize context, send to Gemini CLI | Bash | Full plan + file lists |
| plan-coder | Implement single file | Read, Edit, Write, Glob, Grep, Bash | Status + verified |

## Plan Distribution

Plans are passed directly from `/plan` to `/code` - the orchestrator distributes per-file instructions to coders. This ensures each coder has exactly the instructions it needs.

## Tips

**Getting good results with /plan:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- Include relevant research: "JWT best practices Node.js" not just "JWT"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari"

**Using /code:**
- Copy the Implementation Plan section from `/plan` output
- If some files are BLOCKED, fix issues and re-run `/plan` with a modified task
- Each coder receives its instructions directly

**When things go wrong:**
- BLOCKED status includes error details - read them
- Verify Gemini CLI is installed: `gemini --version`
- Try regenerating the plan with more specific task/research

## Comparison with Other Plugins

| Feature | gemini-swarm | codex-swarm | gemini-pair-pipeline |
|---------|--------------|-------------|----------------------|
| Execution | One-shot | One-shot | Iterative with checkpoints |
| Planning | Gemini CLI (3-flash-preview) | Codex CLI (gpt-5.2) | Gemini CLI (3-flash-preview) |
| User control | Review plan, then execute | Review plan, then execute | Checkpoints during discovery |
| Commands | /plan + /code (separate) | /plan + /code (separate) | /orchestrate (all-in-one) |
| Use case | Well-defined tasks with Gemini | Well-defined tasks with Codex | Exploratory tasks |

Use **gemini-swarm** when you know what you want and want Gemini's architectural planning.

Use **codex-swarm** when you prefer OpenAI's Codex with high reasoning effort.

Use **gemini-pair-pipeline** when you need iterative discovery with user checkpoints.
