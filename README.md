# jimmy-skills

`jimmy-skills` is a manifest-first engineering skill pack for Go and backend-heavy work. The repository is reorganized to follow the pack/domain layout used in PrepKit, with all active skills living under `packs/engineering/skills/domain/`.

This repo is adapted from and still explicitly references selected material from [`samber/cc-skills-golang`](https://github.com/samber/cc-skills-golang). A few remaining domain skills intentionally target Samber-maintained libraries, for example `golang-samber-hot`.

## Layout

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
.claude-plugin/
.cursor-plugin/
gemini-extension.json
```

## Install

Use the repository directly:

```bash
npx skills add https://github.com/jimnguyendev/jimmy-skills --all
```

Or clone it into your local skill discovery path:

```bash
git clone https://github.com/jimnguyendev/jimmy-skills.git ~/.agents/skills/jimmy-skills
```

Codex, Claude Code, Cursor, and other Agent Skills-compatible tools can load the pack from there. Cursor metadata points to `./packs/engineering/skills/domain/`.

## Engineering Pack

The `engineering` pack currently ships 28 Go-focused domain skills covering:

- testing, troubleshooting, observability, performance, and benchmarking
- database, gRPC, CLI, dependency management, and CI
- code style, naming, safety, error handling, and modernization
- project layout, documentation, design patterns, and data structures
- library-specific guidance where it is still useful for this workflow

Quick start commands and pack metadata live in:

- [kit.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/kit.manifest.json)
- [packs/engineering/pack.manifest.json](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/packs/engineering/pack.manifest.json)
- [packs/engineering/pack-quickstart.md](/Users/jimnguyen/workspaces/kits/pro-workflow/tmp/cc-skills-golang/packs/engineering/pack-quickstart.md)

## Conventions

- Cross-skill references should use `jimmy-skills@<skill-name>`.
- New or updated skills belong under `packs/engineering/skills/domain/<skill-name>/`.
- Legacy publisher metadata and distribution automation are intentionally removed from this fork.
- Logging guidance is being standardized around `zap`.

## Attribution

This repository is a curated fork for Jim Nguyen's workflow. It removes the previous distribution and publisher metadata, but it still acknowledges the upstream source and Samber ecosystem references where they remain relevant to the skill content.
