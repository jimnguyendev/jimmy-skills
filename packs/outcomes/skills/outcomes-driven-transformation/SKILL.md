---
name: outcomes-driven-transformation
description: Use this skill when the user needs to drive organizational or internal-team change through outcomes and experiments. Apply it for internal stakeholder alignment, transformation programs, treating colleagues as customers, defining internal behavior change, or using experiments to move a team or organization forward instead of relying on paper plans. For defining product or user outcomes, prefer `outcome-thinking`. For customer-journey roadmaps, prefer `outcomes-based-planning`. For team structure, backlog reset, and trust repair, prefer `organizing-for-outcomes`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Outcomes-Driven Transformation

Use this skill when the target of change is the organization itself: internal habits, internal alignment, or the way teams work together.

## Boundaries

- Use `outcome-thinking` when the subject is user or customer behavior rather than internal adoption.
- Use `outcomes-based-planning` when the main task is to reframe a product roadmap.
- Use `organizing-for-outcomes` when the core issue is team topology, intake, prioritization, or backlog management.

## Default procedure

1. Identify the internal audience whose behavior must change.
2. Treat those colleagues or stakeholders as customers with needs, frictions, and adoption constraints.
3. Define the desired internal behavior change as the outcome.
4. Design a small intervention or experiment that could change that behavior.
5. Observe the response and iterate instead of writing a large theoretical transformation plan.

## Defaults

- Internal transformation still uses outcomes: behavior change first, deliverables second.
- Colleagues are customers when you are asking them to adopt a strategy, tool, or way of working.
- Use experiments for organizational change because behavior change is hard to predict on paper.
- Prefer small adoption tests over broad mandatory rollouts when uncertainty is high.

## Example outcome types

- leaders can explain the strategy consistently
- product and engineering arrive with shared success metrics
- developers propose expected impact, not only implementation details
- teams adopt a new technology or workflow after a measured pilot

## Gotchas

- Internal transformation often fails because it is announced as output, not designed as adoption.
- A strategy is not real if every leader describes it differently.
- Internal behavior change needs its own success signals, not just launch artifacts.
- Large transformation plans without experiments usually hide uncertainty instead of reducing it.

## References

- Read [references/outcomes-driven-transformation.md](references/outcomes-driven-transformation.md) when the task involves internal adoption, transformation strategy, or experimenting toward organizational change.
