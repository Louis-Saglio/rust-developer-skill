# rust-developer-skill

An agent skill that makes a coding agent write, review and refactor Rust by one strict, opinionated rulebook, so the code it produces can be trusted locally without reading the whole codebase.

## What it enforces

- **Errors first.** Every fallible function gets its own error type, defined right before it; no shared error hierarchies. No `unwrap()` / `expect()` in production unless a comment proves the call cannot fail.
- **Invalid states are unrepresentable.** Newtypes instead of bare `String`/`u64` for domain concepts, enums instead of strings and boolean flags, validation in constructors.
- **Pure core, impure edges.** I/O, logging, time and randomness live in top-level functions and adapters; non-trivial projects use hexagonal architecture (ports as traits).
- **Simple over clever.** YAGNI, standard library before crates, long cohesive functions rather than Clean Code fragmentation, `cargo fmt` and `clippy` as hard gates.

When rules conflict, the skill applies an explicit priority order (error correctness, then type safety, then purity, then simplicity). In reviews, it reports rule violations first and proposes a compliant rewrite. A complete worked example is in [`references/compliant-function-example.md`](references/compliant-function-example.md).

## Use

Nothing to invoke: the skill activates whenever the agent writes, reviews or designs Rust code.

This rulebook is deliberately opinionated and departs from mainstream Rust advice on error design, function length and Clean Code. Fork it and edit `SKILL.md` to encode your own rules.

## License

[MIT](LICENSE)
