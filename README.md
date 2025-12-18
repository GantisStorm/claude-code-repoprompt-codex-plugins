# The Pair Planning Framework

A **pluggable multi-agent planning framework** for Claude Code. Swap planning engines (Claude, RepoPrompt, Codex, Gemini) while keeping the same discovery agents and execution patterns.

> **The Core Insight:** All planning engines follow the same meta-pattern: **Discovery → Planning → Execution**. The framework standardizes discovery (code-scout, doc-scout) and execution (plan-coder) while making the planning layer pluggable. This means you can switch between Claude's speed, RepoPrompt's file intelligence, Codex's reasoning depth, or Gemini's stateless simplicity - all using identical workflows.

## Core Concept: Pluggable Planning

```
┌───────────────────────────────────────────────────────────────────────┐
│                     PAIR PLANNING FRAMEWORK                           │
│                                                                       │
│   DISCOVERY (same for all)          PLANNING (choose one)             │
│   ────────────────────────          ─────────────────────             │
│                                                                       │
│   ┌─────────────┐                   ┌─────────────────────────────┐   │
│   │ code-scout  │                   │     Claude    (pair-*)      │   │
│   │ (codebase)  │                   ├─────────────────────────────┤   │
│   └──────┬──────┘    CONTEXT        │   RepoPrompt  (repoprompt-*)│   │
│          │          ─────────►      ├─────────────────────────────┤   │
│   ┌──────┴──────┐                   │     Codex     (codex-*)     │   │
│   │  doc-scout  │                   ├─────────────────────────────┤   │
│   │ (external)  │                   │     Gemini    (gemini-*)    │   │
│   └─────────────┘                   └──────────────┬──────────────┘   │
│                                                    │                  │
│                                                    │ PLAN             │
│                                                    ▼                  │
│                                                                       │
│   EXECUTION (same for all)                                            │
│   ────────────────────────                                            │
│                                                                       │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │
│   │ plan-coder  │  │ plan-coder  │  │ plan-coder  │  (parallel)       │
│   │   file 1    │  │   file 2    │  │   file 3    │                   │
│   └─────────────┘  └─────────────┘  └─────────────┘                   │
└───────────────────────────────────────────────────────────────────────┘
```

**The framework provides:**
- **Shared agents**: code-scout, doc-scout, plan-coder (same implementation across plugins)
- **Pluggable planners**: Swap between Claude, RepoPrompt MCP, Codex MCP, or Gemini MCP
- **Two execution patterns**: Pipeline (iterative) or Swarm (one-shot)

---

## Why Pluggable Planning?

Different planning engines excel at different tasks. Rather than building four separate systems, the Pair Planning Framework **standardizes everything except the planner** - giving you the flexibility to choose the right tool for each task.

| Planning Engine | Strengths | Best For | Key Nuance |
|-----------------|-----------|----------|------------|
| **Claude** (pair-*) | Fast, no dependencies | Quick tasks, standalone | Narrative format, zero setup |
| **RepoPrompt** (repoprompt-*) | Intelligent file selection | Complex refactors | Persistent plans via `chat_id` |
| **Codex** (codex-*) | High reasoning (gpt-5.2) | Difficult architecture | Deep planning with reasoning effort |
| **Gemini** (gemini-*) | Fast, stateless | Well-defined tasks | Fully one-shot, no session history |

**What stays the same across all engines:**
- Discovery agents: code-scout and doc-scout work identically
- Execution agents: plan-coder implements files the same way
- Commands: `/orchestrate` (pipeline) or `/plan` + `/code` (swarm)
- Output format: `files_to_edit`, `files_to_create`, per-file instructions

**What changes per engine:**
- How context is synthesized into a plan (Narrative vs XML)
- Where plans are stored (orchestrator memory vs MCP)
- How plans reach coders (embedded in prompt vs MCP fetch)
- MCP dependencies (none vs required)

---

## Execution Patterns

The framework offers **two execution patterns** that work with any planning engine.

### Pipeline Pattern (Iterative, Human-in-the-Loop)

```
┌────────────────────────────────────────────────────────────────────────┐
│                       ITERATIVE DISCOVERY LOOP                         │
│                                                                        │
│  code-scout ───► CHECKPOINT ───► doc-scout ───► CHECKPOINT ───► ...    │
│       │               │               │               │                │
│       │          User decides    User decides    User decides          │
│       │         "Add research"   "Add more"      "Complete"            │
│       │               │               │               │                │
│       └───────────────┴───────────────┴───────────────┘                │
│                               │                                        │
│                       context_package                                  │
└───────────────────────────────┼────────────────────────────────────────┘
                                │
                                ▼
                     ┌────────────────────┐
                     │      PLANNER       │
                     │  (pluggable MCP)   │
                     └─────────┬──────────┘
                               │
               ┌───────────────┼───────────────┐
               ▼               ▼               ▼
        ┌────────────┐  ┌────────────┐  ┌────────────┐
        │ plan-coder │  │ plan-coder │  │ plan-coder │  (parallel)
        │   file1    │  │   file2    │  │   file3    │
        └────────────┘  └────────────┘  └────────────┘
```

**Characteristics:**
- Single `/orchestrate` command handles entire workflow
- User controls discovery via checkpoints (AskUserQuestion)
- Incrementally add research until context is complete
- Best for: exploratory tasks, unfamiliar codebases, complex features

**Plugins:** `pair-pipeline`, `repoprompt-pair-pipeline`, `codex-pair-pipeline`, `gemini-pair-pipeline`

---

### Swarm Pattern (One-Shot, Fast Execution)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            /plan COMMAND                                │
│                                                                         │
│        task: "Add logout button" | research: "Session handling"         │
│                                    │                                    │
│                    ┌───────────────┴───────────────┐                    │
│                    ▼                               ▼                    │
│              ┌───────────┐                   ┌───────────┐              │
│              │code-scout │                   │ doc-scout │              │
│              │(codebase) │                   │(external) │              │
│              └─────┬─────┘                   └─────┬─────┘              │
│                    │           (parallel)          │                    │
│                    └───────────────┬───────────────┘                    │
│                                    ▼                                    │
│                          ┌─────────────────┐                            │
│                          │     PLANNER     │                            │
│                          │   (pluggable)   │                            │
│                          └────────┬────────┘                            │
│                                   │                                     │
│                                   ▼                                     │
│                        IMPLEMENTATION PLAN                              │
│                        (files + instructions)                           │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                            /code COMMAND                                │
│                                                                         │
│                   plan: [...] or chat_id: [...]                         │
│                                    │                                    │
│                              Parse input                                │
│                                    │                                    │
│                    ┌───────────────┼───────────────┐                    │
│                    ▼               ▼               ▼                    │
│              ┌──────────┐   ┌──────────┐   ┌──────────┐                 │
│              │plan-coder│   │plan-coder│   │plan-coder│                 │
│              │  file1   │   │  file2   │   │  file3   │                 │
│              └────┬─────┘   └────┬─────┘   └────┬─────┘                 │
│                   │    (parallel)│              │                       │
│                   └──────────────┼──────────────┘                       │
│                                  ▼                                      │
│                           RESULTS TABLE                                 │
│                    (file | status | summary)                            │
└─────────────────────────────────────────────────────────────────────────┘
```

**Characteristics:**
- Two separate commands: `/plan` then `/code`
- No checkpoints - review the plan, then execute
- Scouts always run in parallel
- Best for: well-defined tasks, fast execution

**Plugins:** `pair-swarm`, `repoprompt-swarm`, `codex-swarm`, `gemini-swarm`

---

## Planning Engines

### How Plans Flow: Direct vs MCP Fetch

The key architectural difference is **how plans move from planner to coders**.

#### Direct Plan Distribution (pair-*, codex-*, gemini-*)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             ORCHESTRATOR                                │
│                                                                         │
│  ┌───────────┐         ┌─────────────────────────────────────────────┐  │
│  │  PLANNER  │────────►│             FULL PLAN TEXT                  │  │
│  └───────────┘         │    (returned directly to orchestrator)      │  │
│                        └─────────────────────┬───────────────────────┘  │
│                                              │                          │
│                 ┌────────────────────────────┼────────────────────┐     │
│                 │                            │                    │     │
│                 ▼                            ▼                    ▼     │
│       ┌──────────────┐            ┌──────────────┐      ┌──────────────┐│
│       │  plan-coder  │            │  plan-coder  │      │  plan-coder  ││
│       │              │            │              │      │              ││
│       │target: file1 │            │target: file2 │      │target: file3 ││
│       │plan: [instr1]│            │plan: [instr2]│      │plan: [instr3]││
│       │              │            │              │      │              ││
│       │(plan passed  │            │(plan passed  │      │(plan passed  ││
│       │ IN PROMPT)   │            │ IN PROMPT)   │      │ IN PROMPT)   ││
│       └──────────────┘            └──────────────┘      └──────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

**How it works:**
1. Planner returns the full implementation plan to orchestrator
2. Orchestrator parses and extracts per-file instructions
3. Plan-coders receive instructions **embedded in their prompt**
4. No MCP dependency for coders

**Used by:** pair-*, codex-*, gemini-*

---

#### MCP Plan Storage & Fetch (repoprompt-*)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             ORCHESTRATOR                                │
│                                                                         │
│  ┌───────────┐         ┌─────────────────────────────────────────────┐  │
│  │  PLANNER  │────────►│              RepoPrompt MCP                 │  │
│  │           │         │  ┌───────────────────────────────────────┐  │  │
│  │   calls   │         │  │     PLAN STORED IN CHAT SESSION       │  │  │
│  │ context_  │         │  │         chat_id: abc123               │  │  │
│  │  builder  │         │  └───────────────────────────────────────┘  │  │
│  └───────────┘         └──────────────────────┬──────────────────────┘  │
│        │                                      │                         │
│        │ returns: chat_id + file_lists        │                         │
│        ▼                                      │                         │
│  ┌─────────────────┐                          │                         │
│  │  Orchestrator   │                          │                         │
│  │   only knows    │                          │                         │
│  │   chat_id &     │                          │                         │
│  │   file list     │                          │                         │
│  └────────┬────────┘                          │                         │
│           │                                   │                         │
│           ▼                                   ▼                         │
│  ┌─────────────────┐                 ┌─────────────────┐                │
│  │   plan-coder    │                 │   plan-coder    │                │
│  │                 │                 │                 │                │
│  │ chat_id: abc123 │                 │ chat_id: abc123 │                │
│  │ target: file1   │                 │ target: file2   │                │
│  │                 │                 │                 │                │
│  │  FETCHES plan   │                 │  FETCHES plan   │                │
│  │  via mcp__Repo  │                 │  via mcp__Repo  │                │
│  │  Prompt__chats  │                 │  Prompt__chats  │                │
│  └────────┬────────┘                 └────────┬────────┘                │
│           │                                   │                         │
│           ▼                                   ▼                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        RepoPrompt MCP                             │  │
│  │           (each coder fetches from same chat_id)                  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

**How it works:**
1. Planner stores plan in RepoPrompt, returns `chat_id`
2. Orchestrator only receives `chat_id` + file lists
3. Each plan-coder **fetches** its instructions from RepoPrompt MCP
4. Plan persists for review/resumption

**Used by:** repoprompt-*

---

### Planning Engine Comparison

| Aspect | pair-* (Claude) | codex-* (Codex) | repoprompt-* (RepoPrompt) | gemini-* (Gemini) |
|--------|-----------------|-----------------|---------------------------|-------------------|
| **Plan storage** | Orchestrator memory | Orchestrator memory | RepoPrompt MCP | Orchestrator memory |
| **Plan delivery** | In prompt | In prompt | Via `chat_id` fetch | In prompt |
| **Coders need MCP?** | No | No | Yes | No |
| **Planning model** | Claude | gpt-5.2 | RepoPrompt context_builder | gemini-3-flash-preview |
| **Session continuation** | Accumulated context | Accumulated context | `chat_id` | None (fully one-shot) |

---

### The Meta Framework: What's Shared vs What's Different

All four planning engines implement the **same meta framework**. This is what makes them interchangeable:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        THE PAIR PLANNING FRAMEWORK                          │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    SHARED ACROSS ALL ENGINES                        │   │
│   │                                                                     │   │
│   │  • code-scout agent (identical implementation)                      │   │
│   │  • doc-scout agent (identical implementation)                       │   │
│   │  • plan-coder agent (identical implementation*)                     │   │
│   │  • Orchestrator patterns (Pipeline or Swarm)                        │   │
│   │  • Input format: task: | research: | mode:                          │   │
│   │  • Output format: files_to_edit, files_to_create, per-file instrs   │   │
│   │                                                                     │   │
│   │  *repoprompt plan-coder adds MCP fetch capability                   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    DIFFERENT PER ENGINE                             │   │
│   │                                                                     │   │
│   │  • Planner implementation (how context → plan)                      │   │
│   │  • Plan format (Narrative vs XML)                                   │   │
│   │  • Plan delivery (direct vs MCP fetch)                              │   │
│   │  • MCP dependencies (none vs required)                              │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### Planning Engine Nuances

#### Claude (pair-*) - Standalone, Narrative Format

**Philosophy:** Fast, dependency-free planning using Claude's native capabilities.

**Plan Format:** Narrative prose organized by categories:
- Task description, Architecture, Selected Context, Relationships
- Implementation Notes, Ambiguities, Requirements, Constraints

**Key Characteristics:**
- No external MCP required - runs entirely within Claude Code
- Plans passed directly in prompt to coders
- Best balance of simplicity and capability
- Fastest iteration cycle

**When to use:** Default choice for most tasks. Use when you want simplicity, have no MCP setup, or need rapid iteration.

---

#### RepoPrompt (repoprompt-*) - Intelligent Context, XML Format

**Philosophy:** Leverage RepoPrompt's intelligent file selection and context management.

**Plan Format:** XML architectural instructions:
```xml
<task name="..."/>, <architecture>, <selected_context>,
<relationships>, <implementation_notes>, <ambiguities>,
<requirements>, <constraints>
```

**Key Characteristics:**
- Plans stored in RepoPrompt with `chat_id` for persistence
- Coders fetch instructions via MCP (not embedded in prompt)
- `planner-context` agent optimizes workspace selection before continuing
- `command:fetch` allows re-executing existing plans
- Unique `chat_id` based session continuation

**Unique Agents:**
- `planner-context`: Evaluates and optimizes workspace file selection
- `planner-fetch`: Retrieves existing plan by chat_id

**When to use:** Complex refactors where intelligent file selection matters, or when you want persistent plans you can re-execute.

---

#### Codex (codex-*) - High Reasoning, XML Format

**Philosophy:** Leverage OpenAI's gpt-5.2 with high reasoning effort for complex architectural decisions.

**Plan Format:** XML architectural instructions (same structure as RepoPrompt):
```xml
<task name="..."/>, <architecture>, <selected_context>,
<relationships>, <implementation_notes>, <ambiguities>,
<requirements>, <constraints>
```

**Key Characteristics:**
- Uses `gpt-5.2` model with `reasoningEffort: "high"`
- Plans returned directly to orchestrator (not stored in MCP)
- Coders receive instructions embedded in prompt
- Strong architectural reasoning for complex problems

**MCP Configuration:**
```
model: "gpt-5.2"
reasoningEffort: "high"
```

**When to use:** Difficult architectural problems, complex multi-step planning, or when you want a second opinion from a different model family.

---

#### Gemini (gemini-*) - Fast One-Shot, XML Format

**Philosophy:** Fast planning with Gemini's flash model, fully stateless operation.

**Plan Format:** XML architectural instructions (same structure as Codex):
```xml
<task name="..."/>, <architecture>, <selected_context>,
<relationships>, <implementation_notes>, <ambiguities>,
<requirements>, <constraints>
```

**Key Characteristics:**
- Uses `gemini-3-flash-preview` model
- **Fully one-shot** - NO conversation history or session continuation
- Each planning call is completely independent
- Orchestrator manages ALL context explicitly
- Plans returned directly (not stored)

**Critical Difference:** Unlike other engines, Gemini has no memory between calls. The orchestrator must pass complete context every time. This makes it simpler but requires more explicit context management.

**When to use:** Well-defined tasks where you don't need iterative refinement, or when you want fast planning with a different model perspective.

---

### Choosing the Right Engine

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DECISION TREE                                       │
│                                                                             │
│  Need external MCP?                                                         │
│  ├─ No → Use pair-* (simplest, no dependencies)                             │
│  └─ Yes → What do you need?                                                 │
│           ├─ Intelligent file selection → repoprompt-*                      │
│           ├─ High reasoning effort → codex-*                                │
│           └─ Fast, stateless planning → gemini-*                            │
│                                                                             │
│  Task complexity?                                                           │
│  ├─ Simple/well-defined → pair-* or gemini-* (fast)                         │
│  ├─ Complex refactor → repoprompt-* (file selection)                        │
│  └─ Difficult architecture → codex-* (high reasoning)                       │
│                                                                             │
│  Need plan persistence?                                                     │
│  ├─ Yes → repoprompt-* (chat_id based)                                      │
│  └─ No → Any engine works                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture

### Agent Architecture

All plugins share the **same four agent types**. Only the planner implementation differs.

| Agent | Role | Input | Output |
|-------|------|-------|--------|
| **code-scout** | Investigate codebase patterns, conventions | `task:` + `mode:` | `CODE_CONTEXT` |
| **doc-scout** | Fetch external documentation, APIs | `query:` | `EXTERNAL_CONTEXT` |
| **planner** | Synthesize context into implementation plan | Context package | Plan + file lists |
| **plan-coder** | Implement changes for ONE file | File + instructions | `COMPLETE` or `BLOCKED` |

### Agent Tools by Planning Engine

| Agent | pair-* | repoprompt-* | codex-* | gemini-* |
|-------|--------|--------------|---------|----------|
| code-scout | Glob, Grep, Read, Bash | Glob, Grep, Read, Bash | Glob, Grep, Read, Bash | Glob, Grep, Read, Bash |
| doc-scout | Research tools | Research tools | Research tools | Research tools |
| planner | Read, Glob, Grep, Bash | `context_builder` / `chat_send` | `mcp__codex-cli__codex` | `mcp__gemini-cli__ask-gemini` |
| plan-coder | Read, Edit, Write, Bash | + `mcp__RepoPrompt__chats` | Read, Edit, Write, Bash | Read, Edit, Write, Bash |

### Pipeline Agents

Pipeline plugins use specialized planner agents:

| Agent | Purpose |
|-------|---------|
| **planner-start** | Creates initial plan from discovery context |
| **planner-context** | (repoprompt only) Evaluates and optimizes workspace selection before continuing |
| **planner-continue** | Continues existing session with new task/context |
| **planner-fetch** | (repoprompt only) Fetches existing plan by chat_id |

---

## Installation

### Step 1: Add the Marketplace

```bash
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins
```

### Step 2: Install Plugins

```bash
# Standalone (no MCP required)
/plugin install pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install pair-swarm@claude-code-repoprompt-codex-plugins

# RepoPrompt planning
/plugin install repoprompt-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install repoprompt-swarm@claude-code-repoprompt-codex-plugins

# Codex planning
/plugin install codex-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install codex-swarm@claude-code-repoprompt-codex-plugins

# Gemini planning
/plugin install gemini-pair-pipeline@claude-code-repoprompt-codex-plugins
/plugin install gemini-swarm@claude-code-repoprompt-codex-plugins
```

Or enable in `.claude/settings.local.json`:

```json
{
  "enabledPlugins": {
    "pair-pipeline@claude-code-repoprompt-codex-plugins": true,
    "pair-swarm@claude-code-repoprompt-codex-plugins": true,
    "repoprompt-pair-pipeline@claude-code-repoprompt-codex-plugins": true,
    "repoprompt-swarm@claude-code-repoprompt-codex-plugins": true,
    "codex-pair-pipeline@claude-code-repoprompt-codex-plugins": true,
    "codex-swarm@claude-code-repoprompt-codex-plugins": true,
    "gemini-pair-pipeline@claude-code-repoprompt-codex-plugins": true,
    "gemini-swarm@claude-code-repoprompt-codex-plugins": true
  }
}
```

---

## Usage

### Pipeline Plugins (Iterative Discovery)

```bash
# Claude planning (standalone)
/pair-pipeline:orchestrate command:start | task:Add user authentication

# RepoPrompt planning
/repoprompt-pair-pipeline:orchestrate command:start | task:Add user authentication

# Codex planning
/codex-pair-pipeline:orchestrate command:start | task:Add user authentication

# Gemini planning
/gemini-pair-pipeline:orchestrate command:start | task:Add user authentication

# With initial research
/pair-pipeline:orchestrate command:start | task:Add OAuth2 | research:Google OAuth2 best practices

# Continue session with new task
/pair-pipeline:orchestrate command:continue | task:Add password reset
```

### Swarm Plugins (One-Shot)

```bash
# Step 1: Create plan
/pair-swarm:plan task:Add logout button | research:Session handling best practices
/repoprompt-swarm:plan task:Add logout button | research:Session handling
/codex-swarm:plan task:Add logout button | research:Session handling
/gemini-swarm:plan task:Add logout button | research:Session handling

# Step 2: Execute plan
/pair-swarm:code plan:[paste plan from above]
/repoprompt-swarm:code chat_id:[chat_id from /plan]
/codex-swarm:code plan:[paste plan from above]
/gemini-swarm:code plan:[paste plan from above]
```

---

## Requirements

### Base Requirements (All Plugins)

- **Claude Code** - The CLI for orchestration and execution
- **Node.js 18+** - For running Claude Code

### MCP Requirements by Planning Engine

| Planning Engine | MCP Server | Setup |
|-----------------|------------|-------|
| pair-* (Claude) | None | No additional setup |
| repoprompt-* | RepoPrompt MCP | [RepoPrompt App](https://repoprompt.com) |
| codex-* | [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) | See below |
| gemini-* | [jamubc/gemini-mcp-tool](https://github.com/jamubc/gemini-mcp-tool) | See below |

#### Codex MCP Setup

```bash
# Prerequisites
npm i -g @openai/codex  # or: brew install codex
codex login --api-key "your-openai-api-key"

# Install MCP server
claude mcp add codex-cli -- npx -y codex-mcp-server
```

> **Note:** We use [tuannvm/codex-mcp-server](https://github.com/tuannvm/codex-mcp-server) instead of the official server due to a bug with conversation IDs ([Issue #3712](https://github.com/openai/codex/issues/3712)).

#### Gemini MCP Setup

```bash
# Install MCP server
claude mcp add gemini-cli -- npx -y @anthropic/gemini-mcp-tool
```

> **Note:** Gemini MCP is fully one-shot - it has NO conversation history or session continuation. The orchestrator manages all context and passes it explicitly to planners. Uses `gemini-3-flash-preview` model.

---

## Tips for Best Results

### Writing Good Task Descriptions

**Good:**
- "Add logout button that clears session storage and redirects to /login"
- "Fix login button not responding on mobile Safari - touch events not firing"
- "Add JWT authentication with refresh token rotation"

**Bad:**
- "Add logout"
- "Login broken"
- "Add auth"

### Using Research Effectively

Include `research:` when working with:
- External APIs: `research:Stripe API Node.js`
- Unfamiliar libraries: `research:React Query v5 mutations`
- Best practices: `research:JWT refresh token best practices`

### Handling BLOCKED Status

When a plan-coder returns BLOCKED:
1. Read the error details in the status
2. Fix the underlying issue
3. Re-run with `command:continue` (pipeline) or same plan (swarm)

---

## Directory Structure

```
.
├── .claude-plugin/
│   └── marketplace.json          # Plugin registry
├── pair-pipeline/                # Claude planning + Pipeline
│   ├── agents/                   # code-scout, doc-scout, plan-coder, planners
│   ├── commands/                 # orchestrate
│   └── skills/                   # code-quality
├── pair-swarm/                   # Claude planning + Swarm
│   ├── agents/                   # code-scout, doc-scout, plan-coder, planner
│   ├── commands/                 # plan, code
│   └── skills/                   # code-quality
├── repoprompt-pair-pipeline/     # RepoPrompt planning + Pipeline
│   ├── agents/                   # scouts, plan-coder (MCP), planners, planner-context
│   ├── commands/                 # orchestrate
│   └── skills/                   # code-quality, repoprompt-mcps
├── repoprompt-swarm/             # RepoPrompt planning + Swarm
│   ├── agents/                   # scouts, plan-coder (MCP), planner
│   ├── commands/                 # plan, code
│   └── skills/                   # code-quality, repoprompt-mcps
├── codex-pair-pipeline/          # Codex planning + Pipeline
│   ├── agents/                   # scouts, plan-coder, planners
│   ├── commands/                 # orchestrate
│   └── skills/                   # code-quality, codex-mcps
├── codex-swarm/                  # Codex planning + Swarm
│   ├── agents/                   # scouts, plan-coder, planner
│   ├── commands/                 # plan, code
│   └── skills/                   # code-quality, codex-mcps
├── gemini-pair-pipeline/         # Gemini planning + Pipeline
│   ├── agents/                   # scouts, plan-coder, planners
│   ├── commands/                 # orchestrate
│   └── skills/                   # code-quality, gemini-mcps
├── gemini-swarm/                 # Gemini planning + Swarm
│   ├── agents/                   # scouts, plan-coder, planner
│   ├── commands/                 # plan, code
│   └── skills/                   # code-quality, gemini-mcps
└── README.md
```

---

## Troubleshooting

### MCP Connection Issues

**RepoPrompt:**
- Ensure RepoPrompt app is running
- Check that your project is open in RepoPrompt
- Verify MCP server status in RepoPrompt settings

**Codex:**
- Verify installation: `claude mcp list` should show `codex-cli`
- Check auth: `codex login --api-key "your-key"`
- Increase timeout for complex tasks (600000ms recommended)

**Gemini:**
- Verify installation: `claude mcp list` should show `gemini-cli`
- Gemini is one-shot - no session history, context passed in each call

### Plugin Not Found

1. Verify marketplace: `/plugin marketplace list`
2. Check `settings.local.json` has the plugin enabled
3. Restart Claude Code after changes

### Plan-Coder Failures

Common causes:
- Insufficient context from scouts
- Ambiguous plan instructions
- Pre-existing code issues

Solutions:
- Add more research at checkpoints (pipeline)
- Regenerate plan with more specific task/research (swarm)
- Fix issues and use `command:continue`

---
