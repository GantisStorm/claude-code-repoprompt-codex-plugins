# Gemini Pair Pipeline

Coordinates specialized agents to implement complex, multi-file coding tasks with Gemini CLI. Uses **iterative discovery** where users build context incrementally, then Gemini creates detailed architectural plans using the Architect system prompt.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install gemini-pair-pipeline@claude-code-repoprompt-codex-plugins
```

## Requirements

- **Gemini CLI** - Install with `npm install -g @google/gemini-cli` or `brew install gemini-cli` (macOS)
- **Claude Code** - Orchestration, discovery, and execution

## Commands

### command:start - Discovery Loop

Iterative discovery with checkpoints. Best for complex features.

```bash
/gemini-pair-pipeline:orchestrate command:start | task:Add user authentication with JWT tokens
/gemini-pair-pipeline:orchestrate command:start | task:Fix login button not responding on mobile
/gemini-pair-pipeline:orchestrate command:start | task:Add OAuth2 login | research:Google OAuth2 best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> Gemini planning -> execution

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> Gemini planning -> execution

### command:continue - Continue with New Plan

Discovery loop, then create a new plan (builds on previous context).

```bash
/gemini-pair-pipeline:orchestrate command:continue | task:Add password reset flow
/gemini-pair-pipeline:orchestrate command:continue | task:Add rate limiting to the new endpoints
/gemini-pair-pipeline:orchestrate command:continue | task:Add email verification | research:SendGrid API best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> Gemini planning -> execution

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> Gemini planning -> execution

## How It Works

The orchestrator spawns specialized agents via the `Task` tool with `run_in_background: true`, then retrieves results via `TaskOutput`:

1. **Discovery** - code-scout gathers CODE_CONTEXT; doc-scout adds EXTERNAL_CONTEXT (spawned in parallel when `research:` is provided upfront, otherwise optional at checkpoint)
2. **Checkpoints** - User answers clarifying questions, decides when context is complete
3. **Planning** - Gemini CLI (using `gemini-3-flash-preview` model) receives Architect system prompt + CODE_CONTEXT + architectural narrative and creates detailed plan with per-file instructions
4. **Execution** - Coders run in parallel as background tasks, results collected via TaskOutput

**Background execution pattern:**
- Agents spawn with `run_in_background: true`
- Orchestrator uses `TaskOutput task_id: [agent-id]` to retrieve results
- Enables true parallel execution of scouts and coders

| Command | Discovery | Planning |
|---------|-----------|----------|
| `command:start` | Checkpoints + optional research | Gemini (planner-start) |
| `command:continue` | Checkpoints + optional research | Gemini (planner-continue) |

## Architecture Diagram

```
                              ORCHESTRATOR
                                   │
                                   ▼
                        ┌────────────────────┐
                        │  command:start or  │
                        │  continue          │
                        └─────────┬──────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────┐
│              ITERATIVE DISCOVERY LOOP                   │
│              (Human-in-the-loop control)                │
│                                                         │
│  ┌─────────────────┐                                    │
│  │   code-scout    │                                    │
│  └────────┬────────┘                                    │
│           │                                             │
│           │ CODE_CONTEXT                                │
│           ▼                                             │
│  ┌─────────────────┐                                    │
│  │   CHECKPOINT    │                                    │
│  │ (AskUserQuestion│                                    │
│  │ 1. Clarifying Qs│                                    │
│  │ 2. "Add research│                                    │
│  │  or "Complete"  │                                    │
│  └────────┬────────┘                                    │
│           │                                             │
│      ┌────┴────┐                                        │
│      │         │                                        │
│      ▼         │                                        │
│  ┌─────────┐   │                                        │
│  │doc-scout│   │                                        │
│  └────┬────┘   │                                        │
│       │        │                                        │
│       │ EXTERNAL_CONTEXT                                │
│       ▼        │                                        │
│  ┌─────────────────┐                                    │
│  │   CHECKPOINT    │                                    │
│  │ 1. Clarifying Qs│                                    │
│  │ 2. "Add more"   │                                    │
│  │  or "Complete"  │                                    │
│  └───┬────┬────────┘                                    │
│      │    │                                             │
│      │    └───► (loop back)                             │
│      │                                                  │
│      │ context_package                                  │
└──────┼──────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────┐
│  planner-start  │
│       or        │
│ planner-continue│
│(Gemini 3-flash) │
│                 │
└────────┬────────┘
         │
         │ FULL PLAN + file_lists
         ▼
┌─────────────────┐
│   plan-coder    │
│    (1/file)     │
│                 │
│ Input:          │
│  target_file    │
│  action         │
│  plan (direct)  │
└────────┬────────┘
         │
         │ status: COMPLETE/BLOCKED
         ▼
┌─────────────────┐
│  ORCHESTRATOR   │
│ Phase 3 Review  │
│                 │
│ Collects results│
│ Reports to user │
└─────────────────┘
```

**Key flows:**
- `command:start` -> discovery -> planner-start -> plan-coder
- `command:continue` -> discovery -> planner-continue -> plan-coder

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT + clarification |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT + clarification |
| planner-start | Synthesize prompt, create plan via Gemini CLI | Bash | Full plan + file lists |
| planner-continue | Synthesize prompt, create new plan via Gemini CLI | Bash | Full plan + file lists |
| plan-coder | Implement single file | Read, Edit, Write, Glob, Grep, Bash | status + verified |

## Plan Distribution

Plans are passed directly from planners to coders - the orchestrator distributes per-file instructions. This avoids session continuation issues and ensures each coder has exactly the instructions it needs.

## Tips

**Choosing the right command:**
- `command:start` - Explore unfamiliar code, want checkpoints
- `command:continue` - Build on previous context with a new plan

**Getting good results:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari" not "Login broken"
- Use `research:` for external APIs

**When things go wrong:**
- BLOCKED status includes error details - read them
- Re-run with `command:continue` after fixing blockers
- Incomplete context? Add research at checkpoints
- Verify Gemini CLI is installed: `gemini --version`

## Comparison with Other Plugins

| Feature | gemini-pair-pipeline | codex-pair-pipeline | gemini-swarm |
|---------|----------------------|---------------------|--------------|
| Execution | Iterative with checkpoints | Iterative with checkpoints | One-shot |
| Planning | Gemini CLI (3-flash-preview) | Codex CLI (gpt-5.2) | Gemini CLI (3-flash-preview) |
| User control | Checkpoints during discovery | Checkpoints during discovery | Review plan, then execute |
| Commands | /orchestrate (all-in-one) | /orchestrate (all-in-one) | /plan + /code (separate) |
| Use case | Exploratory tasks with Gemini | Exploratory tasks with Codex | Well-defined tasks |

Use **gemini-pair-pipeline** when you need iterative discovery with Gemini's architectural planning.

Use **codex-pair-pipeline** when you prefer Codex's gpt-5.2 with high reasoning effort.

Use **gemini-swarm** when you know what you want and just need fast parallel execution.

## See Also

For RepoPrompt-based planning, see **repoprompt-pair-pipeline** which uses RepoPrompt's context_builder for planning and chat_id for session management.
