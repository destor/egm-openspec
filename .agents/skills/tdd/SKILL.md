---
name: tdd
license: MIT
description: Internal reference for test-driven development at agreed seams, used by the sdd-* skills. The reference for what a good test is, where tests go (seams and the project's seam registry), the anti-patterns (implementation-coupled, tautological, horizontal slicing) and the red → green loop. Use when writing or reviewing tests, when agreeing test seams in sdd-create-spec, when planning tests in sdd-spec-to-tasks, or when implementing an OpenSpec task.
---

<!-- Adapted from github.com/mattpocock/skills skills/engineering/tdd/SKILL.md at 3f59913 (MIT, see LICENSE). Changed: references to the codebase-design and code-review skills removed; OpenSpec note and the Seam registry section added. tests.md and mocking.md are unchanged copies. -->

# Test-Driven Development

TDD is the red → green loop. This skill is the reference that makes that loop produce tests worth keeping: what a good test is, where tests go, the anti-patterns, and the rules of the loop. Every section applies on every cycle: consult them before and during the loop, not after.

When exploring the codebase, read `GLOSSARY.md` (if it exists) so test names and interface vocabulary match the project's domain language, and respect ADRs in the area you're touching.

## What a good test is

Tests verify behavior through public interfaces, not implementation details. Code can change entirely; tests shouldn't. A good test reads like a specification: "user can checkout with valid cart" tells you exactly what capability exists, and it survives refactors because it doesn't care about internal structure.

See [tests.md](tests.md) for examples and [mocking.md](mocking.md) for mocking guidelines.

## Seams: where tests go

A **seam** is the public boundary you test at: the interface where you observe behavior without reaching inside. Tests live at seams, never against internals.

**Test only at pre-agreed seams.** Before writing any test, write down the seams under test and confirm them with the user. No test is written at an unconfirmed seam. You can't test everything, so agreeing the seams up front is how testing effort lands on the critical paths and complex logic instead of every edge case.

Ask: "What's the public interface, and which seams should we test?" Give each proposed seam a one-line note on what it catches and what it misses.

In an OpenSpec change, the pre-agreed seams are the **Testing** decision in `design.md`, and each task names its seam in its test plan (`Verify via`). The approved spec scenarios are the independent source of truth for expected values.

### Seam registry (large repositories)

In a large repository, seams are recorded per area so every change uses the same ones. The project's agent instructions name the registry, for example `Seam registry: docs/testing/seams/index.md`. The index maps paths to area files; each area file names its seams and the boundaries tests may fake:

| Path | Seam file |
| --- | --- |
| `<path glob>` | `<area file>` |

Path not listed, or no registry → global rules only; note the gap.

Global rules:

- Test behaviour caller/user observes through seam. Never shape: field lists, key order, config objects, document ASTs, call order of internal fns.
- Fake only system boundaries listed for the area. Never mock own modules between seam and boundary.
- Do not re-prove what tooling proves. Type checker, linter and code generation already fail on missing/renamed fields, wrong types, lint drift → no test for those.
- One behaviour per test. Spec scenario ≠ one test each; group scenarios that share one observable outcome.
- Refactor without behaviour change breaks test → test wrong. Fix or delete test, not code.
- Security invariants (access, data exposure, cache isolation) are behaviour → test them, at seam named in area file.
- Existing tests violating these rules = legacy. Do not extend pattern; do not rewrite unless task says so.

## Anti-patterns

- **Implementation-coupled**: mocks internal collaborators, tests private methods, or verifies through a side channel (querying the database instead of using the interface). The tell: the test breaks when you refactor but behavior hasn't changed.
- **Tautological**: the assertion recomputes the expected value the way the code does (`expect(add(a, b)).toBe(a + b)`, a snapshot derived by hand the same way, a constant asserted equal to itself), so it passes by construction and can never disagree with the code. Expected values must come from an independent source of truth: a known-good literal, a worked example, the spec.
- **Horizontal slicing**: writing all tests first, then all implementation. Bulk tests verify _imagined_ behavior: you test the _shape_ of things rather than user-facing behavior, the tests go insensitive to real changes, and you commit to test structure before understanding the implementation. Work in **vertical slices** instead: one test → one implementation → repeat, each test a **tracer bullet** that responds to what the last cycle taught you.

## Rules of the loop

- **Red before green.** Write the failing test first, then only enough code to pass it. Don't anticipate future tests or add speculative features.
- **One slice at a time.** One seam, one test, one minimal implementation per cycle.
- **Refactoring is not part of the loop.** It belongs to the review stage, not the red → green implementation cycle.
