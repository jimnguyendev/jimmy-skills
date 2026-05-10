---
name: data-architecture-strategy
description: Use this skill when the user needs to choose or compare data architectures such as RDW, data lake, modern data warehouse, data fabric, lakehouse, or data mesh. Apply it for stage-based architecture decisions, schema-on-read vs schema-on-write trade-offs, self-service BI planning, or deciding what architecture fits a company's maturity and constraints, even if the user does not explicitly say "architecture strategy." For retry safety, backfills, or duplicate writes, prefer `data-pipeline-reliability`. For marts, semantic metrics, and warehouse modeling inside a chosen architecture, prefer `data-engineering`.
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents.
metadata:
  author: jimnguyendev
  version: "1.0.0"
allowed-tools: Read Edit Write Glob Grep Bash(git:*) Agent AskUserQuestion
---

# Data Architecture Strategy

Use this skill when the question is "what architecture should we use, and why now?"

## Boundaries

- Use `data-engineering` when the architecture family is already chosen and the main task is how to model or serve data inside it.
- Use `data-pipeline-reliability` when the main issue is retries, merge semantics, backfills, or recovery behavior.
- Use `prep-data-strategy` for PREP-specific domain rollout and shared-dimension questions.

## Default procedure

1. Identify the company's current maturity, team size, and main users of data.
2. Separate near-term needs from future options.
3. Choose the least complex architecture that satisfies current decisions and response-time needs.
4. Treat architecture choice as a staged roadmap, not a one-shot identity decision.

## Defaults

- Stage 1-2 teams should usually start with a simple RDW or centralized warehouse path.
- Add a data lake or MDW layer only when raw-data staging, ML, or multi-format ingestion clearly requires it.
- Choose lakehouse only if one-repository economics and ML workflows matter more than mature RDW semantics.
- Treat data mesh as an organizational model for very large enterprises, not a default target.
- Prefer schema-on-write when business metrics must be trustworthy early.

## Decision guide

### Quick match

| Primary need | Architecture | Key trade-off |
|-------------|-------------|---------------|
| Mature BI, strong security, easy analyst access | RDW | High cost, compute-storage coupled |
| Cheap raw staging for many formats | Data lake | Not a standalone analytics interface |
| Lake staging + relational serving + ML | MDW | Two systems to manage, data staleness risk |
| Enterprise governance, lineage, MDM | Data fabric | Highest centralized complexity and cost |
| Single repository for analytics + ML | Lakehouse | Weaker security and query perf than RDW |
| Decentralized domain ownership at scale | Data mesh | Organizational shift, 100+ engineers needed |

### MDW vs lakehouse

Choose MDW when dashboards need millisecond response, compliance requires row/column-level security, or most consumers are non-technical. Choose lakehouse when ML and analytics both need first-class access, two-copy cost is a binding constraint, or data staleness is unacceptable.

### Evolution triggers

- Semi-structured or unstructured data at scale: add data lake, move toward MDW.
- ML workloads need file access: add lake for ML (MDW) or evaluate lakehouse.
- Two-copy cost is top budget item: evaluate lakehouse consolidation.
- Central data team is bottleneck across 3+ domains at 100+ engineers: evaluate data mesh.
- Regulatory requirements demand lineage and MDM: add data fabric layers.

## Gotchas

- "Modern" is not a reason to adopt an architecture.
- A lakehouse is not the same thing as an MDW; MDW still replicates some data into an RDW.
- Data mesh solves organizational scaling more than technical novelty.
- Architecture that outruns team maturity becomes delivery drag, not leverage.

## References

- Read [references/architecture-strategy.md](references/architecture-strategy.md) when the task is to compare architectures, stage a roadmap, or justify why one option is overkill.
