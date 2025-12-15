---
name: code-quality
description: Run code quality checks. Use after making code changes, before commits, or when asked to check code quality.
allowed-tools: Bash, Read, Grep, Glob
---

# Code Quality Checks

Run code quality checks after making code changes.

## Commands

```bash
# Full check (type checking + linting)
./start.sh check

# Type checking only
./start.sh typecheck

# Linting only
./start.sh lint

# Linting with auto-fix
./start.sh lint-fix
```

## Workflow

1. Make code changes
2. Run `./start.sh check`
3. If lint errors can be auto-fixed, run `./start.sh lint-fix`
4. Fix remaining errors manually
5. Re-run check to verify
