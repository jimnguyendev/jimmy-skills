# CLAUDE.md

## Repo Purpose

This repository maintains `jimmy-skills` as a stack-first skill kit.

The guiding backend philosophy is:

- organize by business capability, not technical layers
- start with fewer packages and split only when pain appears
- keep names short and avoid repeating package/type context
- keep types close to where they are used
- keep dependencies one-way; use consumer-side interfaces to break cycles

## Structure

```text
kit.manifest.json
packs/
  backend/
    pack.manifest.json
    pack-quickstart.md
    skills/domain/<skill-name>/SKILL.md
  frontend/
    pack.manifest.json
    pack-quickstart.md
    skills/domain/<skill-name>/SKILL.md
  engineering/
    pack.manifest.json
    pack-quickstart.md
.claude-plugin/plugin.json
.cursor-plugin/plugin.json
gemini-extension.json
.github/workflows/
```

## Skill Rules

- Every skill lives under `packs/<pack>/skills/domain/<skill-name>/SKILL.md`.
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
- Use `frontend-core` before framework-specific frontend skills.
- Use `frontend-react` and `frontend-vue` for framework-specific frontend implementation.

## Maintenance

When adding or renaming skills:

1. Update the corresponding `packs/<pack>/pack.manifest.json`.
2. Update `kit.manifest.json` if packs change.
3. Update `README.md` if public routing or pack descriptions change.
4. Re-run the repo validation checks.

## Attribution

The repo is no longer structured as an upstream mirror. Keep upstream acknowledgement minimal and factual, only where reused material or library-specific references still remain.
