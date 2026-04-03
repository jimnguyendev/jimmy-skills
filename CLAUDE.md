# CLAUDE.md

## Repo Purpose

This repository maintains `jimmy-skills`, a manifest-first engineering skill pack for Go/backend work. The repository layout follows the PrepKit pack/domain pattern instead of the older flat `skills/` layout.

## Structure

```text
kit.manifest.json
packs/
  engineering/
    pack.manifest.json
    pack-quickstart.md
    skills/
      domain/
        <skill-name>/
          SKILL.md
          references/
          evals/
          assets/
.claude-plugin/plugin.json
.cursor-plugin/plugin.json
gemini-extension.json
.github/workflows/
```

## Skill Rules

Every skill lives at `packs/engineering/skills/domain/<skill-name>/SKILL.md`.

Required frontmatter fields:

- `name`
- `description`
- `user-invocable`
- `license`
- `compatibility`
- `metadata.author`
- `metadata.version`
- `allowed-tools`

Repo conventions:

- `metadata.author` must be `jimnguyendev`.
- Do not add legacy publisher metadata fields.
- Cross references must use `jimmy-skills@<skill-name>`.
- Keep `SKILL.md` concise and push detail into `references/`.
- If a skill covers a third-party library, include a short disclaimer pointing to official docs.
- Keep evaluations colocated under `evals/`.

## Pack Maintenance

When adding, removing, or renaming a skill:

1. Update `packs/engineering/pack.manifest.json`.
2. Update `README.md` if the public pack description changes.
3. Update plugin manifests if directory paths change.
4. Re-run repo checks or adjust `.github/workflows/validate.yml` if validation rules need to change.

## Naming

- Repository name: `jimmy-skills`
- Pack id: `engineering`
- Skill namespace: `jimmy-skills@<skill-name>`
- GitHub remote: `git@github.com:jimnguyendev/jimmy-skills.git`

## Git / Release

- Do not reintroduce publisher-specific automation tied to the old upstream repository.
- Keep workflows generic and repository-owned.
- Do not auto-bump versions unless explicitly requested.

## Attribution

The repo is adapted from `samber/cc-skills-golang`. Keep the acknowledgement in `README.md`, especially for any surviving Samber-library skills such as `golang-samber-hot`.
