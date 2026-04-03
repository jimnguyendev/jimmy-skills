---
name: golang-error-handling
description: "Idiomatic Golang error handling — creation, wrapping with %w, errors.Is/As, errors.Join, custom error types, sentinel errors, panic/recover, the single handling rule, structured logging with zap, and HTTP request logging middleware. Built to make logs usable at scale with log aggregation 3rd-party tools. Apply when creating, wrapping, inspecting, or logging errors in Go code."
user-invocable: false
license: MIT
compatibility: Designed for Claude Code or similar AI coding agents, and for projects using Golang.
metadata:
  author: jimnguyendev
  version: "1.1.1"
allowed-tools: Read Edit Write Glob Grep Bash(go:*) Bash(golangci-lint:*) Bash(git:*) Agent
---

**Persona:** You are a Go reliability engineer. You treat every error as an event that must either be handled or propagated with context — silent failures and duplicate logs are equally unacceptable.

**Modes:**

- **Coding mode** — writing new error handling code. Follow the best practices sequentially; optionally launch a background sub-agent to grep for violations in adjacent code (swallowed errors, log-and-return pairs) without blocking the main implementation.
- **Review mode** — reviewing a PR's error handling changes. Focus on the diff: check for swallowed errors, missing wrapping context, log-and-return pairs, and panic misuse. Sequential.
- **Audit mode** — auditing existing error handling across a codebase. Use up to 5 parallel sub-agents, each targeting an independent category (creation, wrapping, single-handling rule, panic/recover, structured logging).

> **Community default.** A company skill that explicitly supersedes `jimmy-skills@golang-error-handling` skill takes precedence.

# Go Error Handling Best Practices

This skill guides the creation of robust, idiomatic error handling in Go applications. Follow these principles to write maintainable, debuggable, and production-ready error code.

This skill assumes the project's structured logger is `zap`. If the team later replaces it with an internal logger built on top of `zap`, follow the same field-oriented patterns and log shape.

## Best Practices Summary

1. **Returned errors MUST always be checked** — NEVER discard with `_`
2. **Errors MUST be wrapped with context** using `fmt.Errorf("{context}: %w", err)`
3. **Error strings MUST be lowercase**, without trailing punctuation
4. **Use `%w` internally, `%v` at system boundaries** to control error chain exposure
5. **MUST use `errors.Is` and `errors.As`** instead of direct comparison or type assertion
6. **SHOULD use `errors.Join`** (Go 1.20+) to combine independent errors
7. **Errors MUST be either logged OR returned**, NEVER both (single handling rule)
8. **Use sentinel errors** for expected conditions, custom types for carrying data
9. **NEVER use `panic` for expected error conditions** — reserve for truly unrecoverable states
10. **SHOULD use `zap`** for structured error logging — not `fmt.Println` or `log.Printf`
11. **Attach operational context as structured fields** — request IDs, tenant IDs, and user IDs belong in logs, spans, or custom error types, not in ad-hoc string concatenation
12. **Log HTTP requests** with structured middleware capturing method, path, status, and duration
13. **Use log levels** to indicate error severity
14. **Never expose technical errors to users** — translate internal errors to user-friendly messages, log technical details separately
15. **Keep error messages low-cardinality** — don't interpolate variable data (IDs, paths, line numbers) into error strings; attach them as structured fields instead (via `zap` at the log site, spans, or custom error types) so APM/log aggregators (Datadog, Loki, Sentry) can group errors properly

## Detailed Reference

- **[Error Creation](./references/error-creation.md)** — How to create errors that tell the story: error messages should be lowercase, no punctuation, and describe what happened without prescribing action. Covers sentinel errors (one-time preallocation for performance), custom error types (for carrying rich context), and the decision table for which to use when.

- **[Error Wrapping and Inspection](./references/error-wrapping.md)** — Why `fmt.Errorf("{context}: %w", err)` beats `fmt.Errorf("{context}: %v", err)` (chains vs concatenation). How to inspect chains with `errors.Is`/`errors.As` for type-safe error handling, and `errors.Join` for combining independent errors.

- **[Error Handling Patterns and Logging](./references/error-handling.md)** — The single handling rule: errors are either logged OR returned, NEVER both (prevents duplicate logs cluttering aggregators). Panic/recover design, structured context at the logging boundary, and `zap` integration for APM tools.

## Parallelizing Error Handling Audits

When auditing error handling across a large codebase, use up to 5 parallel sub-agents (via the Agent tool) — each targets an independent error category:

- Sub-agent 1: Error creation — validate `errors.New`/`fmt.Errorf` usage, low-cardinality messages, custom types
- Sub-agent 2: Error wrapping — audit `%w` vs `%v`, verify `errors.Is`/`errors.As` patterns
- Sub-agent 3: Single handling rule — find log-and-return violations, swallowed errors, discarded errors (`_`)
- Sub-agent 4: Panic/recover — audit `panic` usage, verify recovery at goroutine boundaries
- Sub-agent 5: Structured logging — verify `zap` usage at error sites, check for PII in error messages

## Cross-References

- → See `jimmy-skills@golang-observability` for structured logging setup, log levels, and request logging middleware
- → See `jimmy-skills@golang-safety` for nil interface trap and nil error comparison pitfalls
- → See `jimmy-skills@golang-naming` for error naming conventions (ErrNotFound, PathError)

## References

- [zap package](https://pkg.go.dev/go.uber.org/zap)
- [zapcore package](https://pkg.go.dev/go.uber.org/zap/zapcore)
