# Skill Routing Reference

## Routing Decision Tree

```text
  "I need to build/change something"
        │
        ├─ Is it a large feature? (multiple files, new concept, migration)
        │   └─ YES → /design (engineering-design-thinking)
        │              │
        │              ├─ Backend-heavy?
        │              │   ├─ Architecture question → backend-core
        │              │   └─ Go implementation → backend-go-*
        │              │
        │              ├─ MyVocab Go feature?
        │              │   └─ myvocap-backend (feature module template)
        │              │
        │              ├─ Full-stack?
        │              │   └─ backend-core → implementation skills
        │              │
        │              └─ Performance concern?
        │                  └─ engineering-perf-optimization-process → language perf skill
        │
        └─ Is it a small change? (single file, known pattern)
            └─ Route directly to implementation skill
```

## Skill Chaining Patterns

### Pattern 1: New Backend Service

```
/design
  → Gate 1-5 produces design brief
  → backend-core (architecture decisions)
  → backend-go-project-layout (package structure)
  → backend-go-database (if persistence needed)
  → backend-go-testing (test strategy)
  → backend-go-observability (monitoring)
```

### Pattern 2: New MyVocab Feature

```
/design
  → Gate 1-5 produces design brief
  → myvocap-backend (feature module template)
  → backend-go-database (sqlc queries)
  → backend-go-testing (test strategy)
```

### Pattern 3: API Integration

```
/design
  → Gate 1-5 produces design brief
  → engineering-rest-api-design (contract first)
  → backend-core (backend structure)
  → implementation skills
```

### Pattern 4: Performance Optimization

```
engineering-perf-optimization-process
  → Gate 1-5 (targets, hot path, profile, solution, rollback)
  → backend-go-performance (Go-specific optimizations)
  → backend-go-benchmark (measurement)
```

## When to Skip /design

- Bug fixes in existing code
- Adding a test for existing behavior
- Config changes
- Documentation updates
- Renaming or refactoring within a single file
- Any change where the "what" and "how" are already clear
