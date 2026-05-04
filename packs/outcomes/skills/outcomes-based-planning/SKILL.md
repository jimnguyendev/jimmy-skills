---
name: outcomes-based-planning
description: Use this skill when the user needs to plan roadmaps or initiatives around outcomes instead of features. Apply it for customer journey mapping, booster and blocker analysis, hypothesis-driven roadmaps, or converting fixed-scope planning into outcome-centered planning, even if the user does not explicitly say "journey map" or "outcomes-based planning." For defining outcomes and leading indicators themselves, prefer `outcome-thinking`. For org structure, backlog reset, and trust rebuilding, prefer `organizing-for-outcomes`. For internal transformation programs, prefer `outcomes-driven-transformation`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Outcomes-Based Planning

Use this skill when the planning problem is uncertainty: the team knows the result it wants, but not yet the right solution.

## Boundaries

- Use `outcome-thinking` when the task is to define what a good outcome is, distinguish it from output, or choose leading indicators.
- Use `organizing-for-outcomes` when the issue is structural rather than roadmap-oriented.
- Use `outcomes-driven-transformation` when the audience whose behavior must change is internal to the organization.

## Default procedure

1. Treat the business goal as impact, not as the roadmap itself.
2. Map the current user or customer journey from left to right.
3. Identify boosters and blockers at each step:
   - behaviors that predict success
   - behaviors that predict failure
4. Turn the roadmap from a list of promised features into questions or hypotheses about behavior change.
5. Define how each hypothesis will be tested and measured before committing to a large build.

## Defaults

- Plan around systems of outcomes, not isolated features.
- Use customer journey maps as the default visualization when behavior is central.
- Write roadmap items as questions or beliefs to test when uncertainty is high.
- Keep the desired outcome fixed longer than the proposed solution.

## Good roadmap language

- `How might we increase the rate at which buyers and sellers meet early?`
- `We believe that if learners receive feedback sooner, exercise completion will rise.`

Avoid:

- `Build X by Q3`

unless the solution is already proven and low-uncertainty.

## Gotchas

- A long feature roadmap often hides untested assumptions, not true certainty.
- Teams confuse dates with confidence.
- If the journey map does not show real behaviors, the planning discussion will collapse back into feature debates.
- Outcome planning does not remove deadlines; it changes what you promise under uncertainty.

## References

- Read [references/outcomes-based-planning.md](references/outcomes-based-planning.md) when the task involves roadmap reframing, journey mapping, boosters or blockers, or hypothesis-driven planning.
