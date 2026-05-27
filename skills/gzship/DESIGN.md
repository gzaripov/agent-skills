# Design — `gzship` skill

> **Status (2026-05-27):** Name chosen (`gzship`). Home: `gzaripov/agent-skills` monorepo at `skills/gzship/`. Extended on 2026-05-25 to add a **PRD Synthesis** phase, **Domain Terms** in the PRD (per-PRD, no separate `CONTEXT.md`), and **lazy ADR** seeding under `docs/adr/`. Extended again on 2026-05-27 (v2) with course-grade additions: **Quality Targets / Constraints / Stakeholders Consulted** in the PRD; **System Analysis / Risks / Integration & Data Ownership** in the design; optional **Revisit conditions** in ADRs; a new **Phase 6 Plan Synthesis** that produces `plan.md` (vertical-slice tracer bullets) and a new **Phase 7 Plan Review Gate**.

## Core principle

A **Claude-Code-first** skill that carries a single feature from a **BDD scenario → PRD → architecture/design → execution plan → vertical-slice TDD implementation**. Every phase produces exactly one artifact, and **no phase advances until both the navigator (cross-model review) and the developer have approved that artifact.** The flow is outside-in: behavior first, product framing second, architecture third, plan fourth, code last.

## Scope and non-goals

- **In scope:** one feature at a time, from idea to merged-ready code on a feature branch.
- **Non-goals:** opening/merging PRs (that is `babysit-pr`), running the cross-model review engine itself (that is `critique-loop`), multi-feature program planning.
- **Claude-Code-first trade-off:** unlike the other skills in this repo, `gzship` has **no `cursor/` or `codex/` variants**. It deliberately depends on Claude Code subagents (the `Agent` tool) and task tracking. This trades repo-wide portability for richer orchestration — an explicit user decision.
- **Dependencies:** the `critique-loop` skill (cross-model review engine), the `plannotator` CLI (developer-review UI at the phase gates), and optionally the `d2` binary (diagram rendering).

## Skill structure

```
skills/gzship/
  SKILL.md                  # orchestrator: 8 phases, gates, subagent dispatch
  references/
    scenarios.md            # BDD scenario writing — book-derived practices + prior-PRD term grep
    prd.md                  # PRD template (Stakeholders · Quality Targets · Constraints · Domain Terms · User Stories · Decisions · Testing · Scope) + Aliased Terms
    architecture.md         # System Analysis + to-be architecture + D2 diagramming + Integration & Data Ownership + Risks + ADR offer criteria & optional Revisit conditions
    plan.md                 # vertical-slice tracer bullets, AFK/HITL, acceptance criteria, horizontal-slicing forbidden
    implementation.md       # double-loop TDD, per-slice cycle, subagent escalation when plan.md has 5+ slices
```

`SKILL.md` is the orchestrator and stays lean. Each reference file holds the heavy how-to for one phase and is read only when that phase runs.

## The 8 phases

### Phase 0 — Frame & track
Derive a kebab-case `<slug>` from the feature request. Create `docs/features/<slug>/`. Confirm the working directory is a git repo and the current branch is a feature branch (not `main`/`master`). Create a Task/Todo list with one entry per phase and per gate — this list is the "goal tracker" the developer can watch.

### Phase 1 — Discovery & Scenarios (BDD)
1. Dispatch parallel `Explore` subagents to survey the existing codebase: where the feature lands, what already exists, which modules/patterns it touches, what it would conflict with.
2. **Grep prior PRDs for canonical domain terms** (`docs/features/*/prd.md` `## Domain Terms` sections) and reuse them so vocabulary doesn't drift across features.
3. Synthesize the findings.
4. Write `docs/features/<slug>/scenarios.md` — Gherkin-style `Given/When/Then`, following `references/scenarios.md`: declarative not imperative, ubiquitous language, one observable behavior per scenario, scenario outlines for example tables. Include a **"How it lands in the product"** section derived from the survey.

### Phase 2 — PRD Synthesis
Synthesize the conversation, the Phase 1 survey, and `scenarios.md` into `docs/features/<slug>/prd.md` using the template in `references/prd.md`:

- **Problem Statement / Solution** from the user's perspective.
- **`## Stakeholders Consulted`** — one honest line per voice that shaped the PRD. *"Primary developer (solo)"* is valid.
- **`## Quality Targets`** — numeric NFRs (latency p95, RPS, data volume, availability %, durability class), tied to scenarios where applicable. No vague adjectives.
- **`## Constraints`** — explicit rails (stack, deadline, regulatory, budget, team capacity).
- **`## Domain Terms`** — every domain concept the feature touches, defined tightly (one or two sentences each), opinionated, with rejected aliases under `_Avoid_:`. Canonical vocabulary for everything downstream.
- **`## Aliased Terms`** — only when deliberately redefining a prior PRD's term. Silent overload is forbidden.
- **User Stories** tagged with the scenarios in `scenarios.md` that verify them.
- **Implementation Decisions / Testing Decisions / Out of Scope** — no file paths or code snippets.

Do not re-interview the developer here; surface gaps as **Open Questions** for the gate.

### Phase 3 — Product Review Gate
Reviews `scenarios.md` and `prd.md` together — one product framing, gated as one:
1. Run `critique-loop`'s **review-only flow** against both files — the navigator reviews adversarially.
2. Resolve code/scope-level asks directly; surface product-level asks to the developer.
3. **Developer review via Plannotator.** Open each file in Plannotator (`plannotator annotate`) and address every returned annotation. The phase advances only on explicit developer approval of both files.
   Both the navigator verdict and the developer approval are required before Phase 4.

### Phase 4 — Architecture & Design
1. Dispatch parallel subagents to survey the **existing architecture**: current module boundaries, data flow, design patterns already in use, prior art for similar features. Skim existing entries in `docs/adr/` that touch this area.
2. Produce two architecture diagrams — **current** and **proposed** — drafted in both D2 and Mermaid, rendered, the clearer of the two embedded in the design.
3. Write `docs/features/<slug>/design.md`. Required sections (full detail in `references/architecture.md`):
   - **`## System Analysis`** — the as-is (modules in the path, key flow as-is, current load vs. PRD Quality Targets, prior art).
   - **`## Components & interfaces`** + **`## Data flow`** — the to-be.
   - **`## Integration & Data Ownership`** — explicit per-call sync/async/streaming choice and per-entity ownership + consistency model.
   - **`## Alternatives considered`** + **`## Tradeoffs`** + **`## Error handling`** + **`## Risks`** (table: risk · scenario · quality · mitigation) + **`## Assumptions & open questions`**.
   - The implementation slice breakdown is **not** in `design.md` — it is the Phase 6 deliverable.
4. **Write ADRs sparingly.** For any decision that is **hard-to-reverse**, **surprising-without-context**, and **the result of a real trade-off** (all three must hold), write a new entry under `docs/adr/NNNN-<slug>.md` — sequential numbering, lazily create `docs/adr/` if missing. Optional **Revisit conditions** section when the trigger to reopen the decision is nameable. ADRs are committed alongside `design.md` in the same `docs:` commit.

### Phase 5 — Design Review Gate
Same shape as Phase 3: `critique-loop` review-only on `design.md` (and any new ADRs), resolve asks, surface architecture/product tradeoffs to the developer, then **developer review via Plannotator** (`plannotator annotate design.md`) — address every returned annotation, advance only on explicit approval. Both approvals required before Phase 6.

### Phase 6 — Plan Synthesis
Decompose the approved `design.md` into **vertical-slice tracer bullets** and write `docs/features/<slug>/plan.md` using the template in `references/plan.md`. Each slice cuts every layer (schema → API → logic → UI → tests), is independently demoable, and is tagged AFK or HITL. Per slice: title, blocked-by, stories covered (`prd.md`), risks addressed (`design.md`), behavioral description, acceptance criteria checklist. The first slice is the tracer bullet (thinnest end-to-end path proving the system wires up). **Horizontal slicing is forbidden.**

### Phase 7 — Plan Review Gate
Same shape as Phase 5: `critique-loop` review-only on `plan.md`, then `plannotator annotate plan.md`. Both approvals required before Phase 8.

### Phase 8 — Staged Implementation
For each slice in `plan.md`, run the **double-loop TDD** cycle:
1. **Write tests** — acceptance test anchored to the slice's acceptance criteria (outer loop, RED), then unit tests (inner loop, RED).
2. **Write code** — minimal code to green, then refactor.
3. **Iterate with `critique-loop`** — review-only flow on the slice diff; resolve asks.
4. **Proceed** — tick the slice's acceptance-criteria boxes in `plan.md`, mark the task done, move to the next slice.

**Escalation:** if `plan.md` has **5 or more slices**, dispatch each slice to its own fresh subagent (Approach 2) to keep the main context lean. After the final slice, run one `critique-loop` review of the whole feature diff and produce a summary report.

## How reviews work — `critique-loop` integration

`gzship` never re-implements review logic. Every gate calls `critique-loop`'s **review-only flow** against the relevant artifact or diff. The navigator configuration (Codex vs. Cursor, model, reasoning effort) belongs to `critique-loop`; `gzship` only hands it the artifact/diff range. The two phase gates additionally include a **developer review** step, which `critique-loop`'s review-only flow does not provide — `gzship` runs that through **Plannotator** (`plannotator annotate <artifact>`), so the developer annotates the artifact in a browser UI rather than via inline chat. `gzship` addresses every returned annotation and advances only on explicit developer approval.

## Artifacts

```
docs/features/<slug>/
  scenarios.md                      # Phase 1 — BDD scenarios
  prd.md                            # Phase 2 — Stakeholders, Quality Targets, Constraints, Domain Terms (+ Aliased Terms when redefining)
  design.md                         # Phase 4 — System Analysis, Components, Integration & Data Ownership, Risks
  plan.md                           # Phase 6 — vertical-slice tracer bullets with AFK/HITL and acceptance criteria
  diagrams/architecture.d2          # current = base, proposed = D2 scenario
  diagrams/architecture-current.svg
  diagrams/architecture-proposed.svg

docs/adr/                           # Phase 4 — lazily created when the first ADR is needed
  NNNN-<slug>.md                    # sequentially numbered; optional Revisit conditions when nameable
```

All committed with `docs:` conventional commits. ADRs created in Phase 4 are committed together with `design.md` in the same commit. `critique-loop`'s `.critique-loop/` review scratch stays gitignored and is not part of the feature.

## Domain vocabulary — per-PRD, not a central `CONTEXT.md`

`gzship` borrows mattpocock/skills' ubiquitous-language discipline but **does not** introduce a separate `CONTEXT.md` at the repo root. Domain terms live in each PRD's `## Domain Terms` section instead. The trade-off:

- **Why per-PRD.** A central `CONTEXT.md` accumulates indefinitely and rots into a stale dictionary nobody reads. Per-PRD scoping keeps terms tied to the feature that introduced them and dies with the feature if it's retired.
- **Cross-feature consistency** is preserved by Phase 1's prior-PRD grep (`grep -l "^## Domain Terms" docs/features/*/prd.md`): before defining a new term, the skill reads what previous PRDs have already named.
- **Deliberate redefinition** is handled by the PRD's `## Aliased Terms` section, which calls out the prior PRD path and which term is being deprecated.
- **Silent term drift** is explicitly forbidden (Rule 5 in `references/prd.md`).

## D2 handling

`SKILL.md` checks `d2 --version`. If present, render each `.d2` to `.svg`. If missing, offer `brew install d2` (or the official install script); if the developer declines, commit the `.d2` source only and embed it as a fenced code block in `design.md`. D2's `scenarios` keyword lets one file express the current → proposed transition.

## Reference file content (book-derived)

- **`scenarios.md`** — from *BDD in Action* (John Ferguson Smart), *Specification by Example* (Gojko Adzic), *Discovery* (Seb Rose & Gáspár Nagy), *The Cucumber Book* (Wynne & Hellesøy): Three Amigos / Example Mapping, declarative scenarios, ubiquitous language, anti-patterns (imperative or UI-coupled scenarios), scenario outlines. Plus the prior-PRD term-grep discipline (no central `CONTEXT.md`).
- **`prd.md`** — from mattpocock/skills' `to-prd` (PRD template: Problem · Solution · User Stories · Implementation Decisions · Testing Decisions · Out of Scope) and `grill-with-docs` (Domain Terms format adapted from `CONTEXT.md`); plus *Domain-Driven Design* (Eric Evans) for the ubiquitous-language motivation and *Inspired* (Marty Cagan) for "PRD as synthesis, not interview." Quality Targets, Constraints, and Stakeholders Consulted sections adopted from the systems-analysis course (requirements layer: business + functional + non-functional + constraints).
- **`architecture.md`** — from *Working Effectively with Legacy Code* (Michael Feathers): seams, characterization tests, reading existing architecture before proposing changes; plus D2 syntax for architecture diagrams (containers, connections, `scenarios` for before/after) and install/fallback handling; plus mattpocock/skills' ADR offer criteria (hard-to-reverse + surprising + real trade-off) and tight 1–3-sentence ADR template. System Analysis, Risks register, Integration & Data Ownership sections, and ADR Revisit conditions all adopted from the systems-analysis course.
- **`plan.md`** — from mattpocock/skills' `to-issues` (vertical-slice tracer bullets, AFK/HITL, acceptance criteria, blocked-by) and *The Pragmatic Programmer* (Hunt & Thomas) for the tracer-bullet metaphor; plus Kent Beck's vertical-slice TDD lifted from per-test to per-slice level.
- **`implementation.md`** — from *Test-Driven Development by Example* (Kent Beck): red-green-refactor; *Growing Object-Oriented Software, Guided by Tests* (Freeman & Pryce): double-loop / outside-in TDD; plus stage decomposition heuristics and the subagent-per-stage escalation rule.

## Building `gzship` itself

Because `gzship` is a skill, it is built with the `writing-skills` TDD process: baseline pressure scenarios (RED) → write the skill (GREEN) → close loopholes (REFACTOR). Discipline targets — behaviors the skill must make bulletproof:

- skipping the developer-approval gate after a navigator approval;
- advancing a phase on the navigator verdict alone;
- skipping the test-first step within an implementation stage;
- expanding scope mid-stage instead of deferring to the design.
