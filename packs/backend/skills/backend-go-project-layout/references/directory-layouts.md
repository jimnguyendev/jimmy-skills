# Directory Layouts

## Universal Layout (Most Projects)

```
project/
├── cmd/                    # Entry points - ONE subdirectory per main package
│   ├── server/            # Main application #1
│   │   └── main.go
│   ├── client/            # Main application #2
│   │   └── main.go
│   └── migrate/           # Main application #3
│       └── main.go
│   └── cli/               # Main application #4
│       └── main.go
│   └── worker/            # Main application #5
│       └── main.go
├── internal/              # Private application code (`internal/` MUST be used for non-exported packages)
│   ├── users/            # Feature package
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── types.go
│   ├── invoices/         # Another feature package
│   └── platform/         # Shared runtime wiring/config only when needed
├── pkg/                   # Public libraries (optional - only if useful to others)
│   └── logger/
│       └── logger.go
├── api/                   # API definitions (optional)
│   └── openapi.yaml
├── configs/               # Configuration files (optional)
│   └── config.yaml
├── scripts/               # Build/deployment scripts (optional)
├── go.mod
├── go.sum
├── Makefile               # Build automation
├── .gitignore             # Git ignore patterns
├── .golangci.yml          # Linter configuration
├── LICENSE                # License file
└── README.md
```

## Small Projects (Single Binary)

For simple tools, keep it minimal:

```
my-tool/
├── cmd/
│   └── my-tool/
│       └── main.go        # Single main package
├── internal/
│   └── core.go            # Application logic
├── go.mod
├── Makefile               # Build automation (optional but recommended)
├── .gitignore             # Git ignore patterns
├── .golangci.yml          # Linter configuration (optional)
├── LICENSE                # License file (recommended)
└── README.md
```

## Libraries (Reusable Code)

```
my-library/
├── example/               # Example
├── logger/                # Public package
│   ├── logger.go
│   └── logger_test.go
├── internal/
│   └── impl/              # Private implementation details
│       └── core.go
├── go.mod
├── go.sum
├── Makefile               # Build automation
├── .gitignore             # Git ignore patterns
├── .golangci.yml          # Linter configuration
├── LICENSE                # License file
└── README.md
```

**Key points for libraries:**

- Put public API in root-level directories (e.g., `logger/`)
- Use `internal/` for private implementation
- Don't use `cmd/` (unless you have example binaries)

## The cmd/ Directory Convention

**CRITICAL**: All `main` packages must reside in `cmd/`. `cmd/` MUST contain only `main.go` with minimal logic — parse flags, wire dependencies, call `Run()`. NEVER put business logic in `cmd/` — it belongs in `internal/` or `pkg/`.

### Single Application

```
cmd/
└── myapp/
    └── main.go    // package main
```

### Multiple Applications

When you need multiple binaries (e.g., server, CLI tool, migration utility):

```
cmd/
├── server/
│   └── main.go        // Runs the API server
├── client/
│   └── main.go        // CLI client tool
├── worker/
│   └── main.go        // Background worker
└── migrate/
    └── main.go        // Database migration utility
```

Each `main.go`:

- Declares `package main`
- Has its own `func main()`
- Can be built independently: `go build ./cmd/...`

**Building all binaries:**

```bash
go build ./cmd/...        # Build all main packages
go build ./cmd/server     # Build specific binary
```

## Feature-First Layout (Recommended for Services)

Structure code around business capabilities, not technical layers. A feature package should contain the files that feature actually needs, whether that is one file or several:

```
project/
├── cmd/
│   └── api/
│       └── main.go            # Wire dependencies, start server
├── internal/
│   ├── users/                 # Users feature
│   │   ├── handler.go         # HTTP handlers for /users
│   │   ├── service.go         # Business logic
│   │   ├── repository.go      # Data access
│   │   ├── types.go           # User, CreateUserRequest, etc.
│   │   └── routes.go          # Route registration
│   ├── invoices/              # Invoices feature
│   │   ├── handler.go
│   │   ├── service.go
│   │   ├── repository.go
│   │   └── types.go
│   ├── auth/                  # Auth feature
│   │   ├── handler.go
│   │   ├── middleware.go
│   │   ├── service.go
│   │   └── types.go
│   └── shared/                # Cross-cutting only when truly needed
│       ├── middleware.go       # Request logging, recovery, etc.
│       ├── pagination.go
│       └── response.go        # Standard JSON response helpers
├── go.mod
├── go.sum
├── Makefile
├── .gitignore
└── .golangci.yml
```

**Key rules:**

- **Keep types near where they are used** — `users/types.go` defines `User`, not a global `models/` package
- **File names do not repeat the package name** — `users/handler.go`, not `users/user_handler.go`
- **`shared/` is a last resort** — only for code genuinely used by 3+ features
- **Features do not import each other directly** — if `invoice` needs user data, define a small interface in `invoice` and inject the implementation from `main`
- **`platform/` or `shared/` must stay small** — do not move feature code there just because two files use it

**When to split a package:**

- The package has too many unrelated reasons to change (mixed concerns)
- Every small feature change forces edits across many packages (wrong boundaries)
- Two developers frequently conflict on the same files (ownership unclear)

**When NOT to split yet:**

- The project is small and one or two packages cover everything comfortably
- You're splitting "just in case" — wait for real pain

## Common Mistakes to Avoid

### Don't Do This

```
myproject/
├── src/              # Go doesn't use /src (Java pattern)
├── main.go           # Don't put main at root
├── utils/            # Generic package name
├── helpers/          # Generic package name
└── common/           # Generic package name
```

### Do This Instead

```
myproject/
├── cmd/
│   └── myapp/
│       └── main.go   # Main in cmd/
├── internal/
│   ├── users/        # Feature names first
│   └── platform/     # Cross-cutting runtime code only
└── pkg/              # Only if useful to others
```
