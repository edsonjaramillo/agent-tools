---
name: cobra-cli-architecture
description: Build or refactor maintainable Go CLIs with Cobra by separating CLI transport, application orchestration, domain rules, and infrastructure adapters.
---

# Cobra CLI Architecture

Use this skill to keep Cobra-based CLIs maintainable as command count and complexity grow.

## What This Skill Is For

Use this when you need clear boundaries between CLI concerns and business logic.

Use cases:

1. New CLI design with long-term maintainability in mind.
2. Refactoring large command handlers that mix parsing, business logic, and I/O.
3. Introducing testable boundaries around external tools, filesystems, and APIs.

Not a great fit:

1. Very small one-command utilities with no expected growth.
2. Short-lived scripts where architecture overhead is unnecessary.

## Core Architecture (At A Glance)

Layer responsibilities:

1. `cli`: Cobra commands, flags/args parsing, request mapping, output rendering.
2. `app`: orchestration and workflow; coordinates domain and ports.
3. `domain`: business rules and invariants; no framework or I/O coupling.
4. `ports`: interfaces that define required side effects.
5. `adapters`: concrete implementations of ports (filesystem, exec, API, etc.).

Dependency flow:

`cmd -> cli -> app -> (domain + ports) <- adapters`

Non-negotiable rules:

1. `domain` must not import Cobra, `os/exec`, HTTP clients, or concrete adapters.
2. `cli` must not hold business rules; it only validates edge input and maps requests.
3. `app` must not depend on Cobra output or terminal formatting details.
4. `adapters` implement `ports`; they are selected and wired at composition boundaries.

## Recommended Structure

```text
.
├── cmd/
│   └── <app>/main.go
├── internal/
│   ├── cli/
│   │   ├── root.go
│   │   ├── <command>.go
│   │   ├── flags/
│   │   ├── output/
│   │   └── completion/
│   ├── app/
│   │   ├── <usecase>.go
│   │   └── errors.go
│   ├── domain/
│   │   ├── <entity>.go
│   │   ├── <rules>.go
│   │   └── <service>.go
│   ├── ports/
│   │   ├── repository.go
│   │   ├── runner.go
│   │   └── clock.go
│   ├── adapters/
│   │   ├── fs/
│   │   ├── exec/
│   │   └── api/
│   └── config/
│       ├── paths.go
│       └── env.go
├── pkg/ # optional: only for truly reusable public libraries
├── scripts/
├── test/
│   ├── integration/
│   └── e2e/
└── docs/
```

Notes:

1. `domain` is a literal folder name, not a placeholder.
2. Names in angle brackets (for example `<app>`, `<command>`) are placeholders to replace.
3. Start simple with one `internal/domain`; split into subdomains only when pressure appears.

## Command Implementation Recipe

For each command:

1. Define `newXCmd()` in `internal/cli/x.go`.
2. Parse and validate flags and args at the CLI edge.
3. Build a request DTO for `app`.
4. Call one app use case (for example `app.RunX(ctx, req)`).
5. Render output through `internal/cli/output`.

Pseudo-flow:

```go
func newRunCmd(uc app.RunUseCase, out output.Printer) *cobra.Command {
    cmd := &cobra.Command{
        Use: "run",
        RunE: func(cmd *cobra.Command, args []string) error {
            req, err := parseRunRequest(cmd, args)
            if err != nil {
                return err
            }
            res, err := uc.Run(cmd.Context(), req)
            return out.RenderRunResult(res, err)
        },
    }
    return cmd
}
```

Keep command files thin. If a handler grows, move logic to `app`.

## Output, Errors, And Exit Codes

Use one output package for:

1. Levels (`INFO`, `OK`, `WARN`, `ERROR`).
2. Color and no-color behavior.
3. Consistent root error printing.
4. Exit-code mapping from typed errors.

Recommendation:

1. Define typed errors in `app` and `domain`.
2. Map those errors to exit codes in one place (root command or output package).
3. Never scatter exit-code logic across individual commands.

## Shell Completions

Use shell completions to improve UX and reduce typing errors.

Completion behavior rules:

1. Prefer context-aware suggestions (based on flags, args position, and current state).
2. For multi-select inputs, never suggest values already chosen in the same command.
3. Keep completion candidates valid for the current command scope; avoid noisy global suggestions.
4. Ensure completion output stays stable and deterministic for the same input context.

## Domain Rules Centralization

Put invariants in `domain`, such as:

1. Naming and format rules.
2. Validation constraints.
3. State transition rules.
4. Classification logic.

Do not duplicate these rules in command handlers or adapters.

## Testing Strategy

1. Domain tests: table-driven, pure logic, no I/O.
2. App tests: fake ports/mocks to verify orchestration and error mapping.
3. CLI tests: args, flags, request mapping, and output behavior.
4. Integration tests: real adapter wiring with temp dirs or test services.
5. E2E tests: binary-level critical user journeys.

## Refactor Triggers

Refactor when pressure is observable:

1. If two or more commands duplicate orchestration, extract an `app` use case.
2. If rules appear in multiple places, move them to `domain`.
3. If side effects spread across layers, define a `port` and implement an `adapter`.
4. If command files exceed readability, split parsing, mapping, and rendering helpers.

## Practical Defaults

1. Start with fewer packages and split only when responsibility pressure appears.
2. Keep internals private (`internal/`) unless code is truly reusable.
3. Favor explicit DTOs between `cli` and `app` for stability.
4. Preserve stable command UX while refactoring internals.

## Quick Review Checklist

1. No business rule is implemented in `cli` or `adapters`.
2. `domain` imports no framework or I/O packages.
3. Every side effect is behind a `port`.
4. Commands parse/map/dispatch/render without orchestration logic leakage.
5. Exit-code mapping is centralized and typed-error based.
6. Tests exist at domain, app, CLI, and at least one integration/E2E path.
