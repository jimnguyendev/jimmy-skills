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

- reserved for shared process and delivery conventions across stacks

## Conventions

- Cross-skill references use `jimmy-skills@<skill-name>`.
- Backend language skills live under `packs/backend/skills/domain/`.
- Frontend framework skills live under `packs/frontend/skills/domain/`.
- Shared stack-agnostic workflow guidance belongs in `packs/engineering/`.
- Legacy publisher metadata and automation remain removed from this fork.

## Key Files

- [kit.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/kit.manifest.json)
- [packs/backend/pack.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/packs/backend/pack.manifest.json)
- [packs/frontend/pack.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/packs/frontend/pack.manifest.json)
- [packs/engineering/pack.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/packs/engineering/pack.manifest.json)
