---
name: rust-developer
description: Apply whenever generating reviewing or refactoring Rust code or making architectural decisions in Rust projects. Enforces the user's specific coding principles including maximum feasible purity of functions, custom error types unique to each fallible function with no shared error types, hexagonal architecture abstraction for database and network access in non-trivial codebases, complete prohibition of unwrap and expect in production unless a comment demonstrates the call is infallible, strict adherence to YAGNI by implementing only current requirements, preference for longer self-contained functions, rejection of Clean Code principles in favor of direct simple code, replacement of string usage with enums where appropriate, wrapping of native types in custom newtypes for enhanced type safety, avoidance of external dependencies in favor of standard library solutions, mandatory application of cargo fmt and clippy, and design choices that render invalid states impossible to construct.
---

# Rust Developer

## Overview

This skill encodes a specific, opinionated set of Rust development principles preferred by the user. It is activated for any task involving writing new Rust code, reviewing existing code, performing refactoring, designing module or project structure, or providing architectural guidance in Rust. All code suggestions, examples, and reviews must conform to these rules. Where these principles conflict with more conventional or "best practice" Rust advice (such as Clean Code recommendations), these principles take precedence.

## Core Principles

When performing any Rust development work, apply the following rules without exception:

### Functional Purity and Side Effects
- Design functions to be pure to the greatest extent possible. A pure function's output depends solely on its inputs, with no observable side effects, no mutation of external state, and deterministic results for the same inputs.
- Isolate all impure operations (database queries, HTTP requests, file system access, logging, randomness, time queries, etc.) inside the highest-level functions or application entry points. Lower-level functions and business logic must remain pure.
- For non-trivial projects, introduce hexagonal architecture (ports and adapters) to abstract data access and external concerns. Define ports as traits in the core domain; implement adapters in outer layers that interact with concrete databases, APIs, or services. This keeps the domain logic pure and testable.

### Error Handling (Highest Priority)
- Every function that can possibly return an error must have its own dedicated, custom error type. This error type is defined locally (e.g., as an enum or struct inside or next to the function or module) and must not be reusable or returned by any other function in the codebase. There are no global error types or shared error hierarchies.
- Define the error type immediately before the function that returns it. The reader should encounter the error and its variants right before reading the function that produces them.
- Error paths are as important as happy paths. Explicitly handle and propagate errors using `?` or `match`/`if let` as appropriate. Never ignore errors.
- `unwrap()` and `expect()` are forbidden in all production code paths. The sole exception occurs when the developer can prove with certainty that the operation cannot fail under any circumstances. In that exceptional case, place a detailed explanatory comment on the immediately preceding line or on the same line, demonstrating the invariants, preconditions, or type-level guarantees that make failure impossible. Example: `// SAFETY: `foo` is validated earlier and the input is known to be ASCII; the UTF-8 conversion documented in the stdlib cannot fail here.`
- Prefer `Result<T, MySpecificError>` return types. Use `thiserror` or manual `impl std::error::Error` only when truly necessary for a specific function's error; otherwise keep errors simple and local.

### Type Safety and Modeling
- Never pass or return bare native types (`String`, `&str`, `u32`, `bool`, `Vec<T>`, etc.) for domain concepts. Always introduce a newtype wrapper or a domain-specific enum/struct. Example: `struct CustomerId(u64);` or `enum OrderStatus { Pending, Shipped, Delivered }` instead of a string or integer.
- Use the type system to make invalid states unrepresentable. Prefer `enum` for mutually exclusive states over `bool` flags or string tags. Use `Option` and `Result` appropriately; avoid "null object" patterns or sentinel values.
- Replace stringly-typed data with enums wherever a closed set of variants exists. Strings are acceptable only for truly open-ended text such as user-provided names or free-form content.

### Architecture and Bounded Contexts
- Avoid creating generic models or DTOs that are reused indiscriminately across the entire application. Each bounded context or use case should have its own tailored data models. A "User" model in an authentication context is different from a "User" model in a reporting context.
- Keep data read/write operations behind abstractions in hexagonal style for anything beyond trivial scripts. The core logic should not know about concrete storage mechanisms.
- Organize modules and crates around bounded contexts. Keep the crate and module structure simple and directly reflective of the current requirements (YAGNI). Avoid deep module hierarchies or premature crate splitting.

### Code Style and Structure
- Favor functional programming idioms: immutable data by default, iterator chains (`map`, `filter`, `fold`, `try_fold`, `collect`, etc.), combinators, and pipelines. Reduce mutable variables and imperative for-loops to the minimum required.
- Strictly follow YAGNI. Implement only the functionality required by the current task. Do not add extra parameters, generic bounds, traits, or modules "just in case" future needs arise. Resist the urge to build frameworks or extension points prematurely.
- Longer functions are preferred when they keep logically related operations together in one place. A 100-line function that solves a complete sub-problem without forcing the reader to navigate ten 10-line helper functions is often clearer. Only extract a helper when it improves readability or enables reuse within the current scope.
- Reject Clean Code dogma. Excessive extraction of tiny functions, deep call stacks, and "tell don't ask" rituals frequently produce over-abstracted, hard-to-follow code. Write straightforward, direct code that a reader can understand by reading top to bottom.
- Avoid clever abstractions, macros, or proc-macro magic unless they deliver an overwhelming readability or safety benefit for the specific problem. Simple, boring, direct code wins.
- Use `cargo fmt` on every change. Address every `cargo clippy` warning and pedantic suggestion. Treat lints as hard errors during development (`#![deny(clippy::all)]` or equivalent in CI).

### Dependencies
- Reach for the Rust standard library first. Only introduce an external crate when the native solution would be substantially more complex, less safe, or significantly slower to implement correctly. Justify every added dependency in a comment near the `Cargo.toml` entry or in the module documentation.

### Async and Concurrency
- Confine async and concurrent operations (e.g., Tokio, async/await, threads, channels) to top-level handlers and adapters. Core domain logic should remain synchronous and pure where possible. When async is required in a bounded context, keep the impure runtime concerns isolated.

### Additional Practices
- Logging, tracing, and observability calls are impure and must be restricted to top-level functions or dedicated adapter layers. Do not intersperse them in pure domain logic.
- Documentation via rustdoc should be used judiciously for public interfaces; prefer clear, self-documenting code and strategic comments over exhaustive documentation.

### Decision Hierarchy (when principles appear to conflict)
1. Error correctness and explicitness (custom per-function errors, no hidden panics).
2. Type safety and making invalid states impossible.
3. Purity and isolation of side effects.
4. Simplicity and YAGNI (no speculative generality).
5. Readability via longer cohesive functions rather than fragmented small ones.
6. Minimalism in dependencies and abstractions.

Apply these principles consistently. When the user provides code or a task that appears to violate them, point out the violation and propose a corrected version that complies. When asked for a code review, structure feedback around adherence to this skill's rules first, followed by other observations.

## Usage Notes
- This skill is opinionated by design and intentionally diverges from mainstream Rust community recommendations in several areas (notably error design, function length, and attitude toward Clean Code).
- It is intended to produce codebases that are easy to reason about locally, have strong compile-time guarantees, and minimize "magic" and hidden control flow.
- For trivial one-off scripts or prototypes the purity and hexagonal rules may be relaxed, but error handling, newtypes, and avoidance of unwrap remain mandatory.
- A concrete example of a function written in full compliance with these principles (newtypes, dedicated error enum, purity, explicit error handling, self-contained logic) is available in `references/compliant-function-example.md`. Consult it when you need a template for typical domain operations.

## References
- `references/compliant-function-example.md` — Detailed walkthrough of a realistic, principle-compliant function with commentary.
