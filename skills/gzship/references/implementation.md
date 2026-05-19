# Staged TDD implementation — `gzship` Phase 5 reference

## Purpose

Phase 5 turns the approved `design.md` into shipped code, one stage at a time. This
file is the standard that work is held to. Read it before the first stage, then run
the per-stage cycle below for every stage in the design's implementation stage
breakdown. The discipline is non-negotiable: tests are written *before* the code they
test, every stage, no exceptions.

## Double-loop TDD

Implementation runs two nested test loops — *outside-in*.

- **Outer loop — the acceptance test.** Take the stage's BDD scenario from
  `scenarios.md` and express it as a failing acceptance test. This is the loop's RED:
  it states the observable behavior the stage must deliver and stays red until the
  whole stage is done.
- **Inner loop — unit red-green-refactor.** Inside the failing acceptance test, drive
  the implementation with small unit tests: write a failing unit test (RED), write
  the minimal code to pass it (GREEN), then refactor with the tests green (REFACTOR).

The loops nest: you stay in the inner red-green-refactor loop — one unit test, one
slice of code, one refactor at a time — until the outer acceptance test passes. The
acceptance test going green is the signal that the stage is behaviorally complete; it
is the only thing that ends the inner loop.

The double loop is from *Growing Object-Oriented Software, Guided by Tests* (Steve
Freeman & Nat Pryce); red-green-refactor is from *Test-Driven Development by Example*
(Kent Beck).

## The per-stage cycle

Run these steps in order for one implementation stage. The test-first ordering is
mandatory — steps 1 and 2 happen before any production code in step 3.

1. **Acceptance test, RED.** Write the acceptance test for the stage's scenario. Run
   it and watch it fail. (Outer loop.)
2. **Unit test, RED.** Write one unit test for the next slice of behavior. Run it and
   watch it fail. (Inner loop.)
3. **Minimal code, GREEN.** Write the least production code that makes the unit test
   pass. Nothing speculative.
4. **Refactor.** With the tests green, clean up code and tests. Behavior unchanged.
5. **Repeat the inner loop.** Return to step 2 for the next slice, and keep looping
   until the acceptance test from step 1 passes.
6. **Review.** Run `critique-loop`'s review-only flow on the stage diff. Resolve
   code-level asks directly; surface architecture/product asks to the developer.
7. **Proceed.** Mark the stage's task done and move to the next stage.

If you reach for production code before its failing test exists, stop — that is a
skipped step, not a shortcut.

## Stage decomposition

The design's implementation stage breakdown is the input; refine it into stages with
these heuristics:

- **One coherent slice of behavior per stage.** A stage delivers one observable
  capability — ideally one acceptance test — not a grab-bag of unrelated changes.
- **One stage ≈ one logical commit.** Each stage should land as a single, reviewable,
  self-consistent commit boundary.
- **Independently testable.** A stage can be driven red-green on its own, with at
  most stubs/fakes for not-yet-built collaborators.
- **Ordered so each builds on the last.** Sequence stages so every stage depends only
  on stages already completed — no forward references.
- **Small enough to hold in context.** If a stage is too large to keep its tests,
  code, and design slice in mind at once, split it.

## Subagent-per-stage escalation

When the breakdown has **5 or more stages**, dispatch each stage to its own fresh
subagent to keep the controller's context lean.

- **Hand the subagent:** the stage spec from the breakdown; the relevant scenario(s)
  from `scenarios.md`; the `design.md` (components, interfaces, the per-stage cycle to
  follow); and the prior stage's result (what was built, key decisions, the diff or
  commit).
- **Subagent runs the full per-stage cycle** for its stage — acceptance test RED,
  inner loop, `critique-loop` review-only — and reports back what it built, the diff,
  test results, and any deferred observations.
- **The controller verifies before continuing.** Confirm the subagent's tests pass,
  the stage diff matches the stage spec and stays in scope, and the `critique-loop`
  review was run and its asks resolved. Only then mark the stage done and dispatch the
  next subagent. A subagent's self-report does not replace this check.

## Final review

After the last stage:

1. Run one `critique-loop` review of the **whole feature diff**.
2. Handle the verdict exactly like any gate (see the SKILL.md flowchart): resolve
   `CHANGES_REQUESTED` and re-review; surface `BLOCK` to the developer and stop.
3. On APPROVE, produce the summary report — the feature branch is ready for the
   developer to open a PR or merge. (Opening/merging is out of scope — see
   `babysit-pr`.)

## Sources

- *Test-Driven Development by Example* — Kent Beck: red-green-refactor, minimal code
  to green, refactor under green tests.
- *Growing Object-Oriented Software, Guided by Tests* — Steve Freeman & Nat Pryce: the
  double loop, outside-in development, acceptance tests driving unit-level TDD.
