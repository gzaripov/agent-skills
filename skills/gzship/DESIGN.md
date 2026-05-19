# Design — `gzship` skill

> **Status (2026-05-19):** Name chosen (`gzship`). Home: `gzaripov/agent-skills` monorepo at `skills/gzship/`. Design approved; implementation plan pending.

## Core principle

A **Claude-Code-first** skill that carries a single feature from a **BDD scenario → architecture/design → staged TDD implementation**. Every phase produces exactly one artifact, and **no phase advances until both the navigator (cross-model review) and the developer have approved that artifact.** The flow is outside-in: behavior first, architecture second, code last.

## Scope and non-goals

- **In scope:** one feature at a time, from idea to merged-ready code on a feature branch.
- **Non-goals:** opening/merging PRs (that is `babysit-pr`), running the cross-model review engine itself (that is `critique-loop`), multi-feature program planning.
- **Claude-Code-first trade-off:** unlike the other skills in this repo, `gzship` has **no `cursor/` or `codex/` variants**. It deliberately depends on Claude Code subagents (the `Agent` tool) and task tracking. This trades repo-wide portability for richer orchestration — an explicit user decision.

## Skill structure

```
skills/gzship/
  SKILL.md                  # orchestrator: 5 phases, gates, subagent dispatch
  references/
    scenarios.md            # BDD scenario writing — book-derived practices
    architecture.md         # surveying existing code/architecture + D2 diagramming
    implementation.md       # double-loop TDD, stage decomposition, per-stage cycle
```

`SKILL.md` is the orchestrator and stays lean. Each reference file holds the heavy how-to for one phase and is read only when that phase runs.

## The 5 phases

### Phase 0 — Frame & track
Derive a kebab-case `<slug>` from the feature request. Create `docs/features/<slug>/`. Confirm the working directory is a git repo and the current branch is a feature branch (not `main`/`master`). Create a Task/Todo list with one entry per phase and per gate — this list is the "goal tracker" the developer can watch.

### Phase 1 — Discovery & Scenario (BDD)
1. Dispatch parallel `Explore` subagents to survey the existing codebase: where the feature lands, what already exists, which modules/patterns it touches, what it would conflict with.
2. Synthesize the findings.
3. Write `docs/features/<slug>/scenarios.md` — Gherkin-style `Given/When/Then`, following `references/scenarios.md`: declarative not imperative, ubiquitous language, one observable behavior per scenario, scenario outlines for example tables. Include a **"How it lands in the product"** section derived from the survey.

### Phase 2 — Scenario Review Gate
1. Run `critique-loop`'s **review-only flow** against `scenarios.md` — the navigator reviews adversarially.
2. Resolve code/scope-level asks directly; surface product-level asks to the developer.
3. **Developer reviews and explicitly approves.**
   Both the navigator verdict and the developer approval are required before Phase 3.

### Phase 3 — Architecture & Design
1. Dispatch parallel subagents to survey the **existing architecture**: current module boundaries, data flow, design patterns already in use, prior art for similar features.
2. Produce two D2 diagrams — **current** and **proposed** — using D2's `scenarios` keyword so a single `architecture.d2` file expresses both states.
3. Write `docs/features/<slug>/design.md`: components & interfaces, data flow, error handling, tradeoffs considered, and the **implementation stage breakdown**. Embed the rendered diagrams.

### Phase 4 — Design Review Gate
Same shape as Phase 2: `critique-loop` review-only on `design.md`, resolve asks, surface architecture/product tradeoffs to the developer, **developer approves**. Both approvals required before Phase 5.

### Phase 5 — Staged Implementation
For each stage in the design's stage breakdown, run the **double-loop TDD** cycle:
1. **Write tests** — an acceptance/behavior test for the stage's scenario (outer loop, RED), then unit tests (inner loop, RED).
2. **Write code** — minimal code to green, then refactor.
3. **Iterate with `critique-loop`** — review-only flow on the stage diff; resolve asks.
4. **Proceed** — mark the stage's task done, move to the next stage.

**Escalation:** if the design's stage breakdown has **5 or more stages**, dispatch each stage to its own fresh subagent (Approach 2) to keep the main context lean. After the final stage, run one `critique-loop` review of the whole feature diff and produce a summary report.

## How reviews work — `critique-loop` integration

`gzship` never re-implements review logic. Every gate calls `critique-loop`'s **review-only flow** against the relevant artifact or diff. The navigator configuration (Codex vs. Cursor, model, reasoning effort) belongs to `critique-loop`; `gzship` only hands it the artifact/diff range. The two phase gates additionally include a **developer approval** step, which `critique-loop`'s review-only flow does not provide — `gzship` adds that explicitly.

## Artifacts

```
docs/features/<slug>/
  scenarios.md
  design.md
  diagrams/architecture.d2          # current = base, proposed = D2 scenario
  diagrams/architecture-current.svg
  diagrams/architecture-proposed.svg
```

All committed with `docs:` conventional commits. `critique-loop`'s `.critique-loop/` review scratch stays gitignored and is not part of the feature.

## D2 handling

`SKILL.md` checks `d2 --version`. If present, render each `.d2` to `.svg`. If missing, offer `brew install d2` (or the official install script); if the developer declines, commit the `.d2` source only and embed it as a fenced code block in `design.md`. D2's `scenarios` keyword lets one file express the current → proposed transition.

## Reference file content (book-derived)

- **`scenarios.md`** — from *BDD in Action* (John Ferguson Smart), *Specification by Example* (Gojko Adzic), *Discovery* (Seb Rose & Gáspár Nagy), *The Cucumber Book* (Wynne & Hellesøy): Three Amigos / Example Mapping, declarative scenarios, ubiquitous language, anti-patterns (imperative or UI-coupled scenarios), scenario outlines.
- **`architecture.md`** — from *Working Effectively with Legacy Code* (Michael Feathers): seams, characterization tests, reading existing architecture before proposing changes; plus D2 syntax for architecture diagrams (containers, connections, `scenarios` for before/after) and install/fallback handling.
- **`implementation.md`** — from *Test-Driven Development by Example* (Kent Beck): red-green-refactor; *Growing Object-Oriented Software, Guided by Tests* (Freeman & Pryce): double-loop / outside-in TDD; plus stage decomposition heuristics and the subagent-per-stage escalation rule.

## Building `gzship` itself

Because `gzship` is a skill, it is built with the `writing-skills` TDD process: baseline pressure scenarios (RED) → write the skill (GREEN) → close loopholes (REFACTOR). Discipline targets — behaviors the skill must make bulletproof:

- skipping the developer-approval gate after a navigator approval;
- advancing a phase on the navigator verdict alone;
- skipping the test-first step within an implementation stage;
- expanding scope mid-stage instead of deferring to the design.
