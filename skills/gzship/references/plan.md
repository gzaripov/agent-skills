# Writing the plan — `gzship` Phase 6 reference

## Purpose

Phase 6 produces `docs/features/<slug>/plan.md`: the **execution plan** for the
feature, written after the design is approved. The plan decomposes the design
into **vertical-slice tracer bullets** — thin end-to-end paths that each
deliver an observable, independently-demoable behavior.

`plan.md` is reviewed at the **Phase 7 gate**: a cross-model navigator — run
through the `critique-loop` skill (e.g. Codex) — reviews it adversarially, and
the developer must also approve it, before implementation begins. The plan is
the contract the implementation phase executes against.

This file has two parts: **Rules** every plan must obey, and **Recommendations**
applied with judgment. Follow the rules always.

## Rules — always

A plan that breaks one of these fails the gate.

1. **Vertical slices only.** Each slice cuts through **every** layer the feature
   touches — schema, API, domain logic, UI, tests — end to end. A slice that
   touches only one layer (schema-only, UI-only) is horizontal and forbidden.
   See *Why vertical, not horizontal* below.
2. **Each slice is independently demoable.** Once a slice is complete, the
   feature visibly does something it didn't before — a user, a stakeholder, or
   a downstream system can observe the change without scaffolding.
3. **Each slice has acceptance criteria as a checklist.** Concrete, observable,
   verifiable conditions for "done". A reviewer can tick each box without
   reading the implementation.
4. **Each slice traces to PRD stories and design risks.** Tag the User Stories
   it advances; tag the Risks (from `design.md`) it closes or mitigates. A
   slice with no story and no risk has no reason to exist.
5. **Each slice is tagged AFK or HITL.** **AFK** = the slice can be executed
   end-to-end without human judgment mid-flight. **HITL** = the slice needs a
   human decision mid-flight (a product call, a security review, an
   architecture pick the design deliberately deferred). Prefer AFK; surface
   HITL early so it can be scheduled.
6. **Slices are ordered so each builds on the last.** Earlier slices are
   prerequisites for later ones; later slices never silently require code from
   a slice not yet executed. Express dependencies with `Blocked by` if the
   ordering is non-trivial.
7. **No file paths and no code snippets.** Those rot. Describe behavior at the
   user-visible / interface level. *Exception:* a decision-encoding snippet a
   prototype produced (a schema, a state shape, a reducer signature) that the
   slice must respect — trim to the decision-rich parts and label it.
8. **One coherent slice of behavior per slice.** A slice the title needs "and"
   to describe is two slices. If a single acceptance check needs three
   sub-behaviors that fail independently, split.

## Recommendations — apply with judgment

- **First slice = tracer bullet** (from *The Pragmatic Programmer*). The
  thinnest possible end-to-end path through every layer the feature touches,
  even if every layer holds only a stub. It proves the path works and exposes
  integration friction early. Subsequent slices fill in behavior along that
  path.
- **Many thin slices over few thick ones.** Slices are cheaper to review,
  cheaper to revert, and easier to parallelize across days. A slice you can
  hold in your head from acceptance criterion to commit is the right size.
- **One slice ≈ one acceptance test ≈ one logical commit.** A slice that takes
  five commits or three acceptance tests was usually two slices stapled
  together.
- **Surface HITL slices early in the order.** Human gates introduce wait
  time; placing them at the front lets the rest of the plan continue while
  the human side resolves.
- **Slice size: aim 30 minutes – 1 day** of implementation work for a single
  developer. Larger = split. Smaller = combine.
- **Title the slice with the behavior it delivers**, not the file it touches.
  *"Discount code reduces order total"* beats *"Add `applyDiscount` to
  CartService"*.

## Why vertical, not horizontal

**Horizontal slicing** delivers one layer at a time — all the schema, then all
the API, then all the UI. It fails for three reasons:

- **Nothing is demoable until the last layer lands.** The plan is a binary —
  either the whole vertical wires up at the end, or it doesn't.
- **Integration friction lands late.** The hardest bugs are at layer
  boundaries. Horizontal slicing pushes those to the merge of all layers.
- **Tests written ahead of code test imagined behavior, not real behavior.**
  This is the same trap *vertical slice* TDD avoids inside a slice — and the
  trap horizontal plans recreate at the plan level.

**Vertical slicing** delivers one user-visible behavior at a time — schema +
API + logic + UI + tests, all of it, for one narrow scenario. Each slice is a
shippable increment.

```
WRONG (horizontal):
  Slice 1: all schema changes
  Slice 2: all API endpoints
  Slice 3: all UI changes
  Slice 4: all tests

RIGHT (vertical):
  Slice 1: happy-path checkout works end-to-end (thin)
  Slice 2: rejected-card path
  Slice 3: gift-card path
  Slice 4: split-tender path
```

This is the *to-issues* rule from mattpocock/skills, lifted from *The Pragmatic
Programmer*'s tracer-bullet metaphor.

## Plan template

```md
# <feature-name> — Plan

## Tracer-bullet slice

**Slice 1: <one-line behavior title>**
- **Type:** AFK
- **Blocked by:** None — can start immediately
- **Stories covered:** 1, 2 *(from `prd.md`)*
- **Risks addressed:** R1 *(from `design.md` § Risks)*
- **Behavioral description:** <2–3 sentences. What does this slice deliver,
  observed from the outside? No file paths.>
- **Acceptance criteria:**
  - [ ] <observable condition 1>
  - [ ] <observable condition 2>
  - [ ] <observable condition 3>

## Subsequent slices

**Slice 2: <next behavior title>**
- **Type:** AFK
- **Blocked by:** Slice 1
- **Stories covered:** 3
- **Risks addressed:** —
- **Behavioral description:** …
- **Acceptance criteria:**
  - [ ] …
  - [ ] …

**Slice 3: <next behavior — HITL example>**
- **Type:** HITL — needs product call on the refund window
- **Blocked by:** Slice 1
- **Stories covered:** 4, 5
- **Risks addressed:** R3
- **Behavioral description:** …
- **HITL question:** *(what the human must answer, ideally before this slice
  starts; include the deadline and who can answer)*
- **Acceptance criteria:**
  - [ ] …

…
```

The plan is short prose + bullets. Aim for **one screen per slice** at most;
if a slice's description and criteria don't fit, the slice is probably too
big.

## Anti-patterns

| Smell | Why it hurts | Fix |
|---|---|---|
| Horizontal slicing (all-schema, all-API, all-UI…) | Nothing demoable until the end; integration friction lands late | Cut vertically — each slice cuts every layer for one narrow behavior |
| Single-layer slice ("add `users.email_verified` column") | Demoes nothing on its own; a follow-up slice is implicitly required for the demo | Bundle the column into the slice that uses it |
| Slice with no acceptance criteria | "Done" is undefined; reviewer can't gate it | Add a checklist of observable conditions |
| Slice with no story or risk linkage | No reason to exist; suspect scope creep | Tag the PRD story or the design risk it serves, or delete |
| Slices titled by file or function names | The title encodes mechanics, not behavior; the slice will drift as files move | Title with the behavior delivered |
| Slice the title needs "and" to describe | Two behaviors packed together; failure doesn't localize | Split into two slices |
| HITL slices buried in the middle | Mid-plan wait blocks the rest of the work | Move HITL slices forward; resolve the human question early |
| File paths and code snippets in the plan | Both go stale within days of writing | Stay at behavior / interface level; use the design for shape |
| Slices that all overlap one file | The plan is a horizontal slice in disguise — five slices touching one module's internals | Re-examine — are these really separate behaviors, or one behavior dressed up? |

## Subagent escalation

When `plan.md` has **5 or more slices**, Phase 8 (Staged Implementation)
dispatches each slice to its own fresh subagent to keep the controller's
context lean. The plan is the brief: the subagent receives the slice spec, the
relevant scenario(s), `prd.md`, `design.md`, and the prior slice's diff.

For fewer than 5 slices, the main session does everything.

## Sources

- mattpocock/skills' `to-issues` — vertical-slice tracer bullets, AFK vs HITL
  tagging, acceptance criteria per slice, "blocked by" dependency expression.
- *The Pragmatic Programmer* — Andy Hunt & Dave Thomas: the tracer-bullet
  metaphor (thin end-to-end path that proves the system wires up, then thicken).
- *Test-Driven Development by Example* — Kent Beck: the vertical-slice
  rationale (don't outrun your headlights; write the next test, watch it fail,
  pass it, repeat) — same discipline lifted from per-test to per-slice.
- Cucumber Book / *Specification by Example* — scenarios as the unit of
  feature decomposition; each slice realises one or more scenarios.
