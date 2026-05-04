# AGENTS.md

## Repo Purpose

This repository is a **Codex plugin marketplace** containing skill plugins for backend, engineering, and data-platform workflows.

The guiding backend philosophy is:

- organize by business capability, not technical layers
- start with fewer packages and split only when pain appears
- keep names short and avoid repeating package/type context
- keep types close to where they are used
- keep dependencies one-way; use consumer-side interfaces to break cycles

## Structure

```text
.Codex-plugin/
  marketplace.json                        Marketplace catalog (3 plugins)
packs/                                    Each pack = one installable plugin
  backend/
    .Codex-plugin/plugin.json            Plugin manifest (skills: ./skills/)
    pack-quickstart.md
    skills/<skill-name>/SKILL.md
  engineering/
    .Codex-plugin/plugin.json            Plugin manifest (skills: ./skills/)
    pack-quickstart.md
    skills/<skill-name>/SKILL.md
  data-engineer/
    .Codex-plugin/plugin.json            Plugin manifest (skills: ./skills/)
    pack-quickstart.md
    skills/<skill-name>/SKILL.md
```

## Install

```bash
# Add marketplace
/plugin marketplace add jimnguyendev/jimmy-skills

# Install plugins
/plugin install backend@jimmy-skills
/plugin install engineering@jimmy-skills
/plugin install data-engineer@jimmy-skills
```

## Skill Rules

- Every skill lives under `packs/<pack>/skills/<skill-name>/SKILL.md`.
- `metadata.author` must be `jimnguyendev`.
- Cross references must use `jimmy-skills@<skill-name>`.
- Keep `SKILL.md` concise and push detailed material into `references/`.
- If a skill documents a third-party library, include a short disclaimer that official docs remain authoritative.
- Do not reintroduce generic layered API guidance as the default. Feature-first is the repo default unless a skill explicitly scopes an alternative.

Required frontmatter:

- `name`
- `description`
- `user-invocable`
- `license`
- `compatibility`
- `metadata.author`
- `metadata.version`
- `allowed-tools`

## Routing Conventions

- Use `backend-core` before language-specific backend skills when the problem is architectural or operational.
- Use `backend-go-*` for Go implementation details.
- Use `engineering-design-thinking` before starting large features.
- Use `data-engineering` for warehouse/platform design and `outcome-thinking` for outcome-centered planning work.

## Marketplace Architecture

### marketplace.json

Located at `.Codex-plugin/marketplace.json`. Lists all installable plugins with relative path sources pointing directly to packs:

```json
{
  "name": "jimmy-skills",
  "plugins": [
    { "name": "backend", "source": "./packs/backend" },
    { "name": "engineering", "source": "./packs/engineering" },
    { "name": "data-engineer", "source": "./packs/data-engineer" }
  ]
}
```

### Pack = Plugin

Each pack under `packs/` is directly an installable plugin. The key is `"skills": "./skills/"` in plugin.json, which tells Codex to scan the `skills/` subdirectory instead of the default `skills/`.

## Maintenance

When adding or renaming skills:

1. Add the skill to `packs/<pack>/skills/<skill-name>/SKILL.md`.
2. Bump `version` in `packs/<pack>/.Codex-plugin/plugin.json`.
3. Update `.Codex-plugin/marketplace.json` if plugin descriptions change.
4. Update `README.md` if public routing or pack descriptions change.

## Attribution

The repo is no longer structured as an upstream mirror. Keep upstream acknowledgement minimal and factual, only where reused material or library-specific references still remain.
