# Pair Pipeline

Coordinates specialized agents to implement complex, multi-file coding tasks. Uses **iterative discovery with checkpoints** and direct planning.

## Quick Start

```bash
# Add the marketplace from GitHub
/plugin marketplace add GantisStorm/claude-code-repoprompt-codex-plugins

# Install the plugin
/plugin install pair-pipeline@claude-code-repoprompt-codex-plugins
```

## Commands

### command:start - Discovery Loop

Iterative discovery with checkpoints. Best for complex features.

```bash
/pair-pipeline:orchestrate command:start | task:Add user authentication with JWT tokens
/pair-pipeline:orchestrate command:start | task:Fix login button not responding on mobile
/pair-pipeline:orchestrate command:start | task:Add OAuth2 login | research:Google OAuth2 best practices
```

**Flow:** code-scout -> checkpoints -> optional doc-scout -> planning -> execution

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> planning -> execution

### command:continue - Continue Previous Context

Discovery loop using accumulated context from previous run.

```bash
/pair-pipeline:orchestrate command:continue | task:Add password reset flow
/pair-pipeline:orchestrate command:continue | task:Add rate limiting | research:Express rate limiting
```

**Flow:** code-scout (if needed) -> checkpoints -> optional doc-scout -> planning -> execution (uses accumulated context)

**With `research:` provided:** code-scout + doc-scout (parallel) -> checkpoints -> planning -> execution

## How It Works

The orchestrator spawns specialized agents via the `Task` tool with `run_in_background: true`, then retrieves results via `TaskOutput`:

1. **Discovery** - code-scout gathers CODE_CONTEXT; doc-scout adds EXTERNAL_CONTEXT (spawned in parallel when `research:` is provided upfront, otherwise optional at checkpoint)
2. **Checkpoints** - User answers clarifying questions, decides when context is complete
3. **Planning** - planner-start creates implementation plan with per-file instructions
4. **Execution** - plan-coders run in parallel as background tasks, results collected via TaskOutput

**Background execution pattern:**
- Agents spawn with `run_in_background: true`
- Orchestrator uses `TaskOutput task_id: [agent-id]` to retrieve results
- Enables true parallel execution of scouts and coders

**Flow:** code-scout -> checkpoints -> optional doc-scout -> planner-start (or planner-continue) -> plan-coder -> review

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
│                 │
└────────┬────────┘
         │
         │ file_lists + per-file plan
         ▼
┌─────────────────┐
│   plan-coder    │
│    (1/file)     │
│                 │
│ Input:          │
│  target_file    │
│  action         │
│  plan (embedded)│
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

## Agents

| Agent | Purpose | Tools | Output |
|-------|---------|-------|--------|
| code-scout | Investigate codebase | Glob, Grep, Read, Bash | Raw CODE_CONTEXT + clarification |
| doc-scout | Fetch external docs | Any research tools | Raw EXTERNAL_CONTEXT + clarification |
| planner-start | Create implementation plan (new task) | Read, Glob, Grep, Bash | file lists + instructions |
| planner-continue | Create implementation plan (continue mode) | Read, Glob, Grep, Bash | file lists + instructions |
| plan-coder | Implement single file | Read, Edit, Write, Glob, Grep, Bash | status + verified |

## Plan Distribution

Plans are embedded directly in the orchestrator's output - coders receive their per-file instructions directly. This avoids any external dependencies and ensures each coder has exactly the instructions it needs.

For `command:continue`, the accumulated context (CODE_CONTEXT, EXTERNAL_CONTEXT, Q&A) is preserved and passed to the planner.

## Tips

**Getting good results:**
- Be specific: "Add logout button that clears session and redirects to /login" not "Add logout"
- For bugs, describe symptoms: "Login button doesn't respond on mobile Safari" not "Login broken"
- Use `research:` for external APIs

**Using command:continue:**
- Use `command:continue` when adding related features to the same area of code
- The accumulated context (CODE_CONTEXT, EXTERNAL_CONTEXT, Q&A) is preserved
- You can add new research with `command:continue | task:... | research:...`

**When things go wrong:**
- BLOCKED status includes error details - read them
- Re-run with `command:continue` after fixing blockers
- Incomplete context? Add research at checkpoints

## Comparison with Other Plugins

| Feature | pair-pipeline | codex-pair-pipeline | pair-swarm |
|---------|---------------|---------------------|------------|
| Execution | Iterative with checkpoints | Iterative with checkpoints | One-shot |
| Planning | Direct (no MCP) | Codex MCP (gpt-5.2) | Direct (no MCP) |
| User control | Checkpoints during discovery | Checkpoints during discovery | Review plan, then execute |
| Commands | /orchestrate (all-in-one) | /orchestrate (all-in-one) | /plan + /code (separate) |
| Use case | Exploratory tasks, no MCP | Exploratory tasks with Codex | Well-defined tasks |

Use **pair-pipeline** for standalone operation with no external dependencies.

Use **codex-pair-pipeline** when you want Codex's gpt-5.2 architectural planning with iterative discovery.

Use **pair-swarm** when you know what you want and just need fast parallel execution.

## See Also

For RepoPrompt-based planning, see **repoprompt-pair-pipeline** which uses RepoPrompt's context_builder for planning and chat_id for session management.

## Requirements

- **Claude Code** - Orchestration, discovery, and execution
