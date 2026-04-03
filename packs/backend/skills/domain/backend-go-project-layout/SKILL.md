---
name: backend-go-project-layout
description: "Provides a guide for setting up Golang project layouts and workspaces. Use this whenever starting a new Go project, organizing an existing codebase, setting up a monorepo with multiple packages, creating CLI tools with multiple main packages, or deciding on directory structure. Apply this for any Go project initialization or restructuring work."
user-invocable: true
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: jimnguyendev
  version: "1.2.0"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent AskUserQuestion
---

**Persona:** You are a Go project architect. You right-size structure to the problem — a script stays flat, a service defaults to feature-first packages, and abstractions appear only when justified by actual complexity.

# Go Project Layout

## Architecture Decision: Ask First

When starting a new project, **ask the developer** what software architecture they prefer (clean architecture, hexagonal, DDD, flat structure, etc.). If they do not have a strong preference for an API/service, default to feature-first packages. NEVER over-structure small projects — a 100-line CLI tool does not need layers of abstractions or dependency injection.

→ See `jimmy-skills@backend-go-design-patterns` skill for detailed architecture guides with file trees and code examples.

## Dependency Injection: Ask Next

After settling on the architecture, **ask the developer** which dependency injection approach they want: manual constructor injection, or a DI library (google/wire, uber-go/dig+fx), or none at all. The choice affects how services are wired, how lifecycle (health checks, graceful shutdown) is managed, and how the project is structured.

## 12-Factor App

For applications (services, APIs, workers), follow [12-Factor App](https://12factor.net/) conventions: config via environment variables, logs to stdout, stateless processes, graceful shutdown, backing services as attached resources, and admin tasks as one-off commands (e.g., `cmd/migrate/`).

## Quick Start: Choose Your Project Type

| Project Type | Use When | Key Directories |
| --- | --- | --- |
| **CLI Tool** | Building a command-line application | `cmd/{name}/`, `internal/`, optional `pkg/` |
| **Library** | Creating reusable code for others | `pkg/{name}/`, `internal/` for private code |
| **Service** | HTTP API, microservice, or web app | `cmd/{service}/`, `internal/`, `api/`, `web/` |
| **Monorepo** | Multiple related packages/modules | `go.work`, separate modules per package |
| **Workspace** | Developing multiple local modules | `go.work`, replace directives |

## Module Naming Conventions

### Module Name (go.mod)

Your module path in `go.mod` should:

- **MUST match your repository URL**: `github.com/username/project-name`
- **Use lowercase only**: `github.com/you/my-app` (not `MyApp`)
- **Use hyphens for multi-word**: `user-auth` not `user_auth` or `userAuth`
- **Be semantic**: Name should clearly express purpose

**Examples:**

```go
// ✅ Good
module github.com/jdoe/payment-processor
module github.com/company/cli-tool

// ❌ Bad
module myproject
module github.com/jdoe/MyProject
module utils
```

### Package Naming

Packages MUST be lowercase, singular, and match their directory name. → See `jimmy-skills@backend-go-naming` skill for complete package naming conventions and examples.

## Directory Layout

All `main` packages must reside in `cmd/` with minimal logic — parse flags, wire dependencies, call `Run()`. Business logic belongs in `internal/` or `pkg/`. Use `internal/` for non-exported packages, `pkg/` only when code is useful to external consumers.

### Feature-First vs Layer-First (Recommended default for APIs: Feature-First)

For services beyond trivial size, **prefer feature-first layout** over layer-first. Group code by business capability, not by technical role:

```
# ❌ Layer-first — one feature scattered across 5+ folders
internal/
├── handlers/
│   ├── user_handler.go
│   └── invoice_handler.go
├── services/
│   ├── user_service.go
│   └── invoice_service.go
├── repository/
│   ├── user_repo.go
│   └── invoice_repo.go
└── models/
    ├── user.go
    └── invoice.go

# ✅ Feature-first — one feature lives in one place
internal/
├── user/
│   ├── handler.go
│   ├── service.go
│   ├── repository.go
│   ├── types.go
│   └── routes.go
├── invoice/
│   ├── handler.go
│   ├── service.go
│   ├── repository.go
│   └── types.go
└── shared/            # only when truly cross-cutting
    ├── middleware.go
    └── pagination.go
```

**Why feature-first wins at scale:**

- **Locality** — modifying one feature means working mostly in one directory
- **Ownership** — clear boundaries make team ownership and code review easier
- **Circular dependency prevention** — features depend on shared code, not on each other
- **Incremental growth** — start with one package, split into features when pain appears

**Extra rule:** shared packages stay small and boring. Create `shared/`, `platform/`, or `common/` only for truly cross-cutting code, not as a dumping ground for every type used by more than one file.

**When layer-first is acceptable:** Very small services (< 500 lines) or pure CRUD apps where features are thin wrappers around a database.

**Do not over-design too early.** Start with fewer packages than you think you need. Split when pain appears, not before.

See [directory layout examples](references/directory-layouts.md) for universal, small project, library, and feature-first layouts, plus common mistakes.

## Essential Configuration Files

Every Go project should include at the root:

- **Makefile** — build automation. See [Makefile template](assets/Makefile)
- **.gitignore** — git ignore patterns. See [.gitignore template](assets/.gitignore)
- **.golangci.yml** — linter config. See the `jimmy-skills@backend-go-linter` skill for the recommended configuration

For application configuration with Cobra + Viper, see [config reference](references/config.md).

## Tests, Benchmarks, and Examples

Co-locate `_test.go` files with the code they test. Use `testdata/` for fixtures. See [testing layout](references/testing-layout.md) for file naming, placement, and organization details.

## Go Workspaces

Use `go.work` when developing multiple related modules in a monorepo. See [workspaces](references/workspaces.md) for setup, structure, and commands.

## Initialization Checklist

When starting a new Go project:

- [ ] **Ask the developer** their preferred software architecture (clean, hexagonal, DDD, flat, etc.)
- [ ] **Ask the developer** their preferred DI approach — manual wiring, google/wire, uber-go/dig+fx, or none
- [ ] Decide project type (CLI, library, service, monorepo)
- [ ] Right-size the structure to the project scope
- [ ] Choose module name (matches repo URL, lowercase, hyphens)
- [ ] Run `go version` to detect the current go version
- [ ] Run `go mod init github.com/user/project-name`
- [ ] Create `cmd/{name}/main.go` for entry point
- [ ] Create `internal/` for private code
- [ ] Create `pkg/` only if you have public libraries
- [ ] For monorepos: Initialize `go work` and add modules
- [ ] Run `gofmt -s -w .` to ensure formatting
- [ ] Add `.gitignore` with `/vendor/` and binary patterns

## Circular Dependencies

Go enforces that package imports form a DAG — no cycles allowed. If package A imports B, then B cannot import A.

**Why Go prohibits them:** faster compilation, cleaner architecture, simpler maintenance.

**Three solutions when you hit a cycle:**

1. **Separate concerns** — move the function to the package where it logically belongs (e.g., stock checking belongs in `inventory`, not `product`)
2. **Merge packages** — if two packages are inseparably intertwined, consolidate them into one
3. **Use interfaces at the consumer side** — define a small interface where you need it, inject the concrete implementation from `main` or a wiring package

**Prevention:**

- Keep packages small and focused on one business capability (feature-first layout helps naturally)
- Maintain one-way dependency direction across layers
- If two features start depending on each other, extract the shared concept into a separate package or define consumer-side interfaces
- Do not solve cycles by adding more technical-layer packages; that usually scatters one concern across more folders and worsens locality
- If two packages are inseparable, merging them is often better than preserving a bad boundary

→ See `jimmy-skills@backend-go-design-patterns` for interface-based decoupling and dependency injection patterns.

## Related Skills

→ See `jimmy-skills@backend-go-cli` skill for CLI tool structure and Cobra/Viper patterns. → See `jimmy-skills@backend-go-linter` skill for golangci-lint configuration. → See `jimmy-skills@backend-go-continuous-integration` skill for CI/CD pipeline setup. → See `jimmy-skills@backend-go-design-patterns` skill for architectural patterns.
