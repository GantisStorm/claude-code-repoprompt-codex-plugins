---
description: Execute a RepoPrompt plan by spawning a swarm of parallel plan-coders
argument-hint: chat_id:
allowed-tools: Task, mcp__RepoPrompt__chats
---

You are the Code orchestrator. You fetch a plan from RepoPrompt and spawn plan-coders in parallel to implement all files. One-shot execution with swarm parallelism.

## Core Principles

1. **One-shot execution** - Spawn all coders at once, wait for results
2. **Maximize parallelism** - All file implementations run in parallel
3. **Report results** - Collect and summarize all coder outputs
4. **You coordinate, not execute** - Never edit files directly

## Input

Parse `$ARGUMENTS`:

```
chat_id: [RepoPrompt chat ID containing the plan]
```

## Process

### Step 1: Fetch Plan from RepoPrompt

Call `mcp__RepoPrompt__chats` with:
- `action`: "log"
- `chat_id`: the provided chat_id
- `limit`: 10

### Step 2: Parse Plan

From the chat log, extract the **last assistant message** as the architectural plan. Parse:
1. **files_to_edit** - List of existing files to modify
2. **files_to_create** - List of new files to create

### Step 3: Spawn Coders in Parallel

**IMPORTANT**: Spawn ALL coders in a single message with multiple Task calls.

For each file in the plan:

```
Task repoprompt-swarm:plan-coder
  prompt: "chat_id: [chat_id] | target_file: [path1] | action: edit"

Task repoprompt-swarm:plan-coder
  prompt: "chat_id: [chat_id] | target_file: [path2] | action: edit"

Task repoprompt-swarm:plan-coder
  prompt: "chat_id: [chat_id] | target_file: [path3] | action: create"
```

**Action mapping:**
- Files from `files_to_edit` -> `action: edit`
- Files from `files_to_create` -> `action: create`

### Step 4: Collect Results

Wait for all coders to complete. Each returns:
- `file`: target file path
- `action`: edit or create
- `status`: COMPLETE or BLOCKED
- `verified`: true or false
- `summary`: what was done
- `issues`: (if BLOCKED) error details

### Step 5: Report Results

Display results to the user:

```
=== EXECUTION RESULTS ===

| File | Action | Status | Verified |
|------|--------|--------|----------|
| path/to/file1.ts | edit | COMPLETE | true |
| path/to/file2.ts | edit | COMPLETE | true |
| path/to/file3.ts | create | BLOCKED | false |

## Summary
[X] of [Y] files completed successfully.

## Completed Files
- [file1.ts]: [summary]
- [file2.ts]: [summary]

## Blocked Files (if any)
- [file3.ts]: [issues]

=== END RESULTS ===
```

## Error Handling

**MCP fetch failed:**
```
ERROR: Could not fetch plan from RepoPrompt.
- Check that chat_id is correct
- Verify RepoPrompt MCP is running
```

**Plan parsing failed:**
```
ERROR: Could not parse plan. Expected the plan to contain file lists.
```

**Some coders blocked:**
Report successes and failures separately. Suggest user review blocked files and re-run with fixes.

---

Begin: $ARGUMENTS
