---
name: backend-core
description: "Shared backend engineering conventions for service design, API boundaries, persistence, async workflows, release safety, and cross-service contracts. Use when the task is backend but not language-specific yet, or when a request spans API design, database changes, messaging, rollout, or production behavior across multiple stacks."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Backend Core

Use this skill before dropping into a language-specific backend skill when the main question is architectural or operational rather than syntax-level.

## Focus Areas

1. API contracts before handler code
2. Zero-downtime schema evolution before migration implementation
3. Idempotency and retries for async flows
4. Authentication and authorization boundaries
5. Observability, rollout, and rollback requirements

## Routing

- For Go implementation details, move to the relevant `jimmy-skills@backend-go-*` skill.
- For frontend integration work, coordinate with `jimmy-skills@frontend-core` plus the relevant framework skill.
- For cross-team delivery or review workflow, use the `engineering` pack guidance.
