---
name: frontend-core
description: "Shared frontend engineering conventions for UI architecture, data fetching boundaries, accessibility, design system integration, rendering strategy, and browser-facing performance. Use when a request is frontend but not tied to a single framework yet, or when it spans component structure, state ownership, API contracts, or UX constraints."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Frontend Core

This skill defines the shared frontend baseline before routing into framework-specific guidance.

## Focus Areas

1. Contract-first data flow with backend services
2. State ownership and cache invalidation boundaries
3. Accessibility, responsive behavior, and loading states
4. Performance budgets for rendering and bundle growth
5. Design-system and component layering decisions

## Routing

- Use `jimmy-skills@frontend-react` for React-specific implementation details.
- Use `jimmy-skills@frontend-vue` for Vue-specific implementation details.
- Pair with `jimmy-skills@backend-core` when the main problem is frontend/backend contract design.
