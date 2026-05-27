# Staged TDD implementation — `gzship` Phase 8 reference

## Purpose

Phase 8 turns the approved `plan.md` into shipped code, one **slice** at a time,
with tests written first. Read this file before the first slice, then run the
per-slice cycle for every slice in `plan.md`. Every slice's diff — and the whole
feature at the end — is reviewed by a cross-model navigator, run through the
`critique-loop` skill (e.g. Codex); see *The per-slice cycle* and *Final review*.

`plan.md` is the contract this phase executes against: each slice's acceptance
criteria are the bar for "done", each slice's behavioral description is what the
tests target, and each slice's `Blocked by` field gates ordering. The design
phase produced the architecture this code lives inside; the plan phase produced
the order. This phase produces the code.

It has two parts: **Rules** every slice must obey, and **Recommendations**
applied with judgment.

## Guiding principle

> *The more your tests resemble the way your software is used, the more confidence
> they give you.* — Kent C. Dodds

Test the feature the way a real caller exercises it. Confidence per unit of effort
is highest for **integration** tests — that is the default; see Rule 2.

## Rules — always

A slice that breaks one of these is not done.

1. **Read the repo's existing tests first.** Match the framework, directory layout,
   fixtures, naming, and assertion style already in use. Do not introduce a second,
   parallel testing style.
2. **Default to integration tests.** Exercise the feature the way it is really used
   — through real collaborators, or high-fidelity fakes for awkward dependencies
   (e.g. testcontainers for a database, a temp dir for the filesystem). Reserve
   isolated unit tests for pure, dependency-free logic (simple models and
   algorithms with no database, network, or filesystem).
3. **Write the test before the code, and watch it fail first** — for the right
   reason (the behavior is absent, not a typo or compile error). Code written
   before its failing test exists is a skipped step; delete it and start the test.
4. **Never derive a test's expected value from the code under test.** Compute it
   independently — from the scenario, by hand, or a known oracle. Importing a
   production constant to form the expectation is a tautological test and verifies
   nothing.
5. **Assert observable behavior through the public interface** — return values,
   errors, visible effects. Never assert implementation internals (private state,
   call sequences, internal structure).
6. **One behavior per test; the test's name says which.** A failure should name the
   broken behavior without reading the test body.
7. **Tests are deterministic, isolated, and order-independent.** No dependence on
   clock, timezone, locale, unseeded randomness, network, or another test's
   leftover state. Inject a seam for every nondeterministic dependency.
8. **No control-flow logic in a test; every test has an assertion that can fail.**
   Refactor only while tests are green. Never present a skipped or known-red test
   as if it passed.

## Recommendations — apply with judgment

- **Mock only true external boundaries.** Prefer real collaborators or fakes; reach
  for a mock only at an unmanaged out-of-process seam. A test that is hard to set
  up is feedback that the design is too coupled — fix the design, don't paper over
  it with mocks.
- **Structure each test Arrange-Act-Assert** — keep the three parts distinct; the
  *Act* is a single identifiable step.
- **One logical assertion (one concept) per test** — several physical asserts that
  jointly describe one outcome are fine.
- **Name tests so a failure reads as a spec** — unit + scenario + expected result.
- **Write minimally-passing tests** — the simplest input that exercises the
  behavior; set only the fields the behavior needs.
- **Prefer per-test data builders over shared setup** — readable, no shared-fixture
  coupling.
- **Keep the suite fast; segregate slow tests** into a separate integration run.
- **Take small red-green steps** — 1–10 minutes each; if green takes longer, the
  step was too big.
- **Use coverage to find untested behavior, not as a target number.**

## Double-loop TDD

Implementation runs two nested loops — *outside-in*.

- **Outer loop — the acceptance test.** Express the slice's BDD scenario as a
  failing acceptance test. It states the observable behavior the slice must deliver
  and stays red until the slice is done.
- **Inner loop — unit red-green-refactor.** Inside the failing acceptance test,
  drive the implementation with small tests: write a failing test (RED), write the
  minimal code to pass it (GREEN), refactor under green (REFACTOR).

Stay in the inner loop — one test, one slice of code, one refactor — until the
outer acceptance test passes. That is the signal the slice is behaviorally complete.

## The per-slice cycle

Run these in order for one slice. Steps 1–2 happen before any production code.

1. **Acceptance test, RED.** Write the acceptance test for the slice's scenario;
   run it and watch it fail.
2. **Unit/inner test, RED.** Write one test for the next slice of behavior; run it
   and watch it fail.
3. **Minimal code, GREEN.** Write the least production code that passes the test.
4. **Refactor.** With tests green, clean up code and tests; behavior unchanged.
5. **Repeat the inner loop** from step 2 until the acceptance test passes.
6. **Review.** Run `critique-loop`'s review-only flow on the slice diff. Resolve
   code-level asks directly; surface architecture/product asks to the developer.
7. **Proceed.** Mark the slice's task done and move on.

## Slice execution — driven by `plan.md`

This phase does **not** decompose the feature into slices — that is `plan.md`'s
job, already approved at the Phase 7 gate. This phase executes them.

Sanity-check each slice before starting, against the heuristics that should
already hold in `plan.md` (see `references/plan.md`):

- **One coherent piece of behavior per slice** — ideally one acceptance test.
- **One slice ≈ one logical commit** — a single reviewable, self-consistent change.
- **Independently testable** — drivable red-green on its own, with at most
  stubs/fakes for not-yet-built collaborators.
- **Ordered so each builds on the last** — no forward references.
- **Small enough to hold in context** — if not, split it.

## Subagent-per-slice escalation

When the breakdown has **5 or more slices**, dispatch each slice to its own fresh
subagent to keep the controller's context lean.

- **Hand the subagent** the slice spec, the relevant scenario(s), `design.md`, and
  the prior slice's result (what was built, key decisions, the diff).
- **The subagent runs the full per-slice cycle** and reports what it built, the
  diff, test results, and any deferred observations.
- **The controller verifies before continuing** — tests pass, the diff matches the
  slice spec and stays in scope, the `critique-loop` review was run and resolved.
  A subagent's self-report does not replace this check.

## Final review

After the last slice:

1. Run one `critique-loop` review of the **whole feature diff**.
2. Handle the verdict like any gate — resolve `CHANGES_REQUESTED` and re-review;
   surface `BLOCK` to the developer and stop.
3. On APPROVE, produce the summary report. The branch is ready for the developer to
   PR or merge (out of scope — see `babysit-pr`).

## Test smells

| Smell | Symptom | Fix |
|---|---|---|
| Tautological test | Expected value comes from the code under test | Compute the expected value independently — spec, by hand, oracle |
| Testing the mock | Assertions only re-state the double's setup | Assert the SUT's real behavior; don't assert on stubs |
| Fragile / overcoupled test | Breaks on refactors that don't change behavior | Assert observable behavior via the public interface |
| Eager test | One test exercises several behaviors | One behavior per test |
| Assertion roulette | Many asserts, no messages — unclear which failed | Split into focused tests, or add failure messages |
| Mystery guest | Depends on an external/undeclared resource | Make every input explicit and local to the test |
| Slow test | Hits network/DB/filesystem/sleep needlessly | Use a fast fake; move genuine integration to a separate run |
| Flaky / order-dependent | Depends on clock, RNG, or another test's state | Inject a seam (clock/RNG); each test arranges its own state |
| Logic in test | `if`/loop/`try` drives the assertions | Replace with parameterized / table-driven tests |
| Vacuous test | Exercises code but asserts nothing falsifiable | Add a real assertion — or delete the test |

## Sources

- *Test-Driven Development by Example* — Kent Beck: red-green-refactor, minimal code
  to green, refactor under green.
- *Growing Object-Oriented Software, Guided by Tests* — Freeman & Pryce: the double
  loop, outside-in, "listen to the tests".
- Kent C. Dodds — the Testing Trophy, "Write tests. Not too many. Mostly
  integration.", and "the more your tests resemble the way your software is used…"
  ([kentcdodds.com/blog/write-tests](https://kentcdodds.com/blog/write-tests)).
- Vladimir Khorikov, *Unit Testing Principles, Practices, and Patterns*; Martin
  Fowler, "Mocks Aren't Stubs": mock only unmanaged out-of-process dependencies.
- The FIRST properties (Fast, Isolated, Repeatable, Self-validating, Timely).
