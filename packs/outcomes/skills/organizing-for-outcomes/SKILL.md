---
name: organizing-for-outcomes
description: Use this skill when the user needs to organize teams, intake, prioritization, collaboration, or stakeholder management around outcomes instead of feature silos. Apply it for team topology, cross-functional workflow, backlog reset, trust rebuilding, defining "done" for outcomes, or shifting from handoff-driven delivery to collaborative outcome work. For defining outcomes or OKRs, prefer `outcome-thinking`. For roadmap reframing and journey maps, prefer `outcomes-based-planning`. For internal change programs and adoption experiments, prefer `outcomes-driven-transformation`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Organizing for Outcomes

Use this skill when the management problem is structural: the team may understand outcomes conceptually, but the organization still runs on feature silos, handoffs, and backlog politics.

## Boundaries

- Use `outcome-thinking` when the problem is conceptual clarity about outcomes, impacts, indicators, or OKRs.
- Use `outcomes-based-planning` when the main task is roadmap conversion or journey mapping.
- Use `outcomes-driven-transformation` when the focus is organizational adoption experiments or internal behavior change programs.

## Default procedure

1. Identify how the organization is currently grouped:
   - by channel
   - by platform
   - by page or feature area
   - by customer journey or outcome
2. Determine which customer or user outcomes cut across those boundaries.
3. Align intake, prioritization, and collaboration around those outcomes.
4. Define how success will be reviewed over time instead of asking "is the feature done?"
5. Rebuild trust with stakeholders through clearer prioritization and honest capacity management.

## Defaults

- Organize around customer journeys or outcomes when cross-functional behavior change matters.
- Use early design-dev-user collaboration instead of design-to-dev handoff.
- Establish hypotheses and review windows before starting major outcome work.
- Reset impossible backlogs instead of maintaining false hope.
- Maintain a separate run-the-business lane for low-uncertainty operational work.

## Signals that reorganization is needed

- teams compete for the same shared resources
- stakeholders negotiate for slots instead of aligning on outcomes
- large backlogs create false expectations
- teams can report feature completion but not value delivered

## Gotchas

- A feature-centric structure will quietly de-prioritize cross-journey outcomes.
- Stakeholders often ask for certainty they cannot actually have; you need review logic, not fake precision.
- Throwing away backlog items can be correct if the backlog is already impossible.
- Without trust, outcome language sounds like an excuse for not delivering.

## References

- Read [references/organizing-for-outcomes.md](references/organizing-for-outcomes.md) when the task involves team structure, cross-functional workflow, backlog reset, or stakeholder trust.
