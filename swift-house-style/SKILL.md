---
name: swift-house-style
description: Expert guidance for Swift code quality and refactoring. Use when developers ask for Swift best-practice reviews, style cleanup, readability improvements, API design feedback, optional/error-handling guidance, performance-minded refactors, or production-readiness checks.
tags: [swift, style, best-practices, refactor, review]
---

# Swift House Style (Best Practices)

When writing or editing Swift, follow these rules unless the user explicitly requests otherwise.

## Core principles
- Prefer **clarity over cleverness**. Optimize for the next reader.
- Keep functions **small and single-purpose**; aim for < 40 lines.
- Make illegal states unrepresentable: use **enums**, **value types**, and **non-optional** properties where possible.
- Prefer **immutability**: `let` first; mutate in tight scopes only.
- Favor **composition** over inheritance.

## Naming & API design
- Use Swift API Design Guidelines: names should form **grammatical phrases** at call sites.
- Avoid abbreviations unless universally known (URL, ID, JSON).
- Prefer argument labels that clarify intent: `add(item:)`, `move(from:to:)`.
- Avoid boolean parameter traps (`doThing(true)`). Prefer enums or clearly labeled parameters.

## Optionals & error handling
- Avoid optional-chaining pyramids. Use `guard let` to unwrap early.
- Avoid force unwrap (`!`) except in tests or truly impossible cases, with a comment.
- Prefer `throws` for recoverable failures; use `Result` when you must store/compose outcomes.
- Define error types (`enum FooError: Error`) with meaningful cases.

## Control flow
- Prefer early exits with `guard`.
- Avoid deeply nested `if`s; extract functions or use `switch`.
- Prefer `switch` for enums; include a `default` only when forward compatibility is required and justified.

## Types, protocols, and generics
- Use protocols to model capabilities, not buckets of unrelated methods.
- Prefer generic constraints that are minimal and readable.
- Avoid over-engineering with generics; if a simple concrete type is clearer, choose clarity.

## Value vs reference
- Prefer structs for models and pure data.
- Use classes for shared mutable identity or when required by frameworks.
- If using classes, document ownership and mutation expectations.

## Performance & memory
- Avoid unnecessary allocations in hot paths; otherwise, write clear code first.
- Use `lazy` only when it measurably helps or prevents expensive work.
- Be careful with capturing `self` in escaping closures; prefer `[weak self]` where appropriate.

## Documentation & comments
- Comments should explain **why**, not **what**.
- Use doc comments for public APIs and non-obvious behavior.
- Keep TODOs actionable and owned (include context, not just “fix later”).

## Review checklist output
When asked to review/refactor Swift code, produce:
1) Issues found (grouped: correctness, concurrency, style, performance)
2) Suggested diff or rewritten code
3) Brief rationale per change