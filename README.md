# jimmy-skills

`jimmy-skills` is a manifest-first multi-pack skill kit organized by stack instead of by one language bucket.

This repo is adapted from and still explicitly references selected material from [`samber/cc-skills-golang`](https://github.com/samber/cc-skills-golang). Some backend skills still intentionally point at Samber-maintained libraries, for example `backend-go-samber-hot`.

## Pack Layout

```text
kit.manifest.json
packs/
  backend/
    pack.manifest.json
    pack-quickstart.md
    skills/domain/
      backend-core/
      backend-go-*/
  frontend/
    pack.manifest.json
    pack-quickstart.md
    skills/domain/
      frontend-core/
      frontend-react/
      frontend-vue/
  engineering/
    pack.manifest.json
    pack-quickstart.md
```

## Installation

```bash
npx skills add https://github.com/jimnguyendev/jimmy-skills --all
```

Or clone locally:

```bash
git clone https://github.com/jimnguyendev/jimmy-skills.git ~/.agents/skills/jimmy-skills
```

Cursor metadata currently points at the backend domain directory because that is the mature implementation pack in this repo today.

## Packs

Backend:

- `backend-core` for shared backend concerns such as API contracts, schema changes, async workflows, rollout, and service boundaries
- `backend-go-*` for Go-specific implementation skills
- currently 29 backend skills: 1 shared backend skill and 28 Go-specialized skills

Frontend:

- `frontend-core` for shared UI architecture and browser-facing concerns
- `frontend-react` and `frontend-vue` as framework entry points
- currently a starter pack intended to grow as you add framework-specific depth

Engineering:

- `engineering-rest-api-design` for cross-stack REST API design conventions, documentation templates, and review checklists
- shared process and delivery conventions across stacks

## Design Philosophy

### Shared-first routing

Skills are never accessed in isolation. The system enforces a routing pattern where architectural concerns are resolved before implementation details:

- Architectural/operational problem → `backend-core` → then `backend-go-*`
- UI architecture problem → `frontend-core` → then `frontend-react` or `frontend-vue`

This forces the AI agent to reason at the right level of abstraction before writing code.

### Knowledge graph over document silos

Every skill cross-references related skills via `jimmy-skills@<skill-name>` convention. For example `backend-go-database` references `backend-go-security` for SQL injection concerns, and `backend-go-concurrency` references `backend-go-context` for cancellation patterns. The result is a navigable knowledge graph rather than disconnected documents.

### Progressive disclosure

Each `SKILL.md` stays concise. Detailed material lives in `references/` subdirectories — the agent reads them only when needed, preserving context window budget. For example `backend-go-security` has 11 reference files covering threat modeling, injection, cryptography, cookies, secrets, and more.

### Multi-mode operation

Skills define distinct behavioral modes (Write, Review, Audit, Debug) so the same domain knowledge adapts to different tasks. Audit mode typically spawns 3-5 parallel sub-agents to scan large codebases efficiently.

### Persona-anchored responses

Each skill assigns a specific engineering persona ("Senior Go security engineer", "Go concurrency engineer — every goroutine is a liability until proven necessary"). This anchors the AI to a clear technical stance instead of generic answers.

### Selective deep thinking

Critical domains (security audits, performance optimization) activate `ultrathink` mode — the system knows where shallow analysis is dangerous and forces deeper reasoning.

## Strengths

| Strength | Detail |
|---|---|
| Cross-IDE portable | Same skill set runs on Claude Code, Cursor, and Gemini via separate plugin descriptors |
| Tool whitelisting | Each skill declares `allowed-tools`, controlling which tools the AI may use |
| Eval-driven quality | `evals.json` files provide concrete test cases (50+ for security alone) to measure skill effectiveness |
| Production-ready assets | Ships with real templates: `.golangci.yml` (33+ linters), Makefile, CI configs |
| Parallel audit scaling | Audit mode distributes work across sub-agents for large codebase analysis |
| Manifest-driven structure | All routing and organization lives in JSON manifests, enabling automated validation and tooling |
| Extensible by design | The empty `engineering` pack and the shared-then-specific routing pattern allow growth without architectural changes |

## Conventions

- Cross-skill references use `jimmy-skills@<skill-name>`.
- Backend language skills live under `packs/backend/skills/domain/`.
- Frontend framework skills live under `packs/frontend/skills/domain/`.
- Shared stack-agnostic workflow guidance belongs in `packs/engineering/`.
- Legacy publisher metadata and automation remain removed from this fork.

## Key Files

- [kit.manifest.json](kit.manifest.json)
- [packs/backend/pack.manifest.json](packs/backend/pack.manifest.json)
- [packs/frontend/pack.manifest.json](packs/frontend/pack.manifest.json)
- [packs/engineering/pack.manifest.json](packs/engineering/pack.manifest.json)
