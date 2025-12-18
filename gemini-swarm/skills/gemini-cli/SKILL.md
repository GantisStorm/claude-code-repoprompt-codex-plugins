---
name: gemini-cli
description: Reference for Gemini CLI usage patterns. Consult before calling gemini via Bash.
---

# Gemini CLI Reference

Quick reference for Gemini CLI commands.

## Basic Command

```bash
gemini "[prompt]" -m gemini-3-flash-preview -o text 2>&1
```

## Common Flags

| Flag | Purpose |
|------|---------|
| `-m gemini-3-flash-preview` | Model selection |
| `-o text` | Human-readable output |
| `-o json` | Structured output with stats |
| `-r [index]` | Resume session by index |
| `--allowed-tools` | Restrict available tools |
| `--list-sessions` | List available sessions |

## Session Continuation

```bash
# List sessions
gemini --list-sessions

# Resume by index
echo "follow-up prompt" | gemini -r 1 -m gemini-3-flash-preview -o text 2>&1
```

## Bash Execution Notes

- Use `dangerouslyDisableSandbox: true` for Bash calls
- Always append `2>&1` to capture all output
- Use timeout of 300000ms (5 min) or longer for complex tasks

## Troubleshooting

**EPERM errors**: Gemini needs write access to `~/.gemini/tmp/` - use `dangerouslyDisableSandbox: true`

**File access**: Gemini can only read files in the workspace directory (project root)

**Rate limits**: Free tier is 60/min, 1000/day. CLI auto-retries with backoff.

## More Information

- CLI reference: `gemini --help`
- Official docs: https://github.com/google-gemini/gemini-cli
