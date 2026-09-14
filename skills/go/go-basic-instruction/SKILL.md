---
name: go-basic-instruction
description: Enforce basic Go code quality rules. Use when writing, editing, or reviewing Go code to apply consistent formatting and linting with gofumpt/go fmt and golangci-lint.
---

# Go Basic Instruction

Follow these rules for all Go code changes.

## Formatting

Run formatting after edits:

```bash
gofumpt -l -w .
```

## Linting

Run lint checks with:

```bash
golangci-lint run ./...
```
