# gzship — Design Spec

**Date:** 2026-05-25
**Status:** Draft, awaiting user approval
**Author:** Brainstormed with Claude (Opus 4.7) on `gzship-skill` branch

## Summary

`gzship` is a Claude-Code-first skill that walks a single feature from a fresh idea to merged code through seven gated phases: discover, PRD, review-A, design, review-B, slice, implement. Each phase produces a committed artifact in `docs/features/<slug>/` (or `docs/adr/` for ADRs). Adversarial reviews are delegated to the existing `critique-loop` skill. Architecture is illustrated with D2 diagrams (current vs. proposed). Implementation runs as vertical-slice tracer bullets with strict per-slice RED→GREEN TDD; subagents fan out for parallel codebase surveys and (when a feature has 5+ slices) for per-slice implementation.

The skill adopts conventions popularised by `mattpocock/skills` (`CONTEXT.md` glossary, `docs/adr/`, the PRD template, vertical-slice tracer bullets, deep-module vocabulary) but **does not depend on `mattpocock/skills` being installed** — the relevant rules are inlined and missing repo files (`CONTEXT.md`, `docs/adr/`) are seeded lazily on first use. The one hard external dependency is `critique-loop` (same repo), which powers every review gate.

## Goals

- One coherent flow for shipping a feature end-to-end, with hard review gates that prevent skipping ahead.
- Every phase produces a durable, PR-reviewable artifact committed under `docs/`.
- Reviews are adversarial and cross-model (via `critique-loop`), then the developer approves.
- Implementation discipline: vertical slices only; per-slice RED→GREEN; no horizontal "all tests then all code".
- Architecture is visible: current and proposed states rendered as D2 diagrams committed alongside the design.
- Vocabulary is consistent: every doc uses the consumer repo's `CONTEXT.md` glossary; deep-module vocabulary (`module / interface / seam / adapter / depth / leverage / locality`) is used when discussing architecture.

## Non-goals

- Not a portable skill. This is **Claude-Code-first** — it uses subagents (`Agent` tool) and the harness task list (`TaskCreate`). No `cursor/` or `codex/` variants. (This is a deliberate departure from the rest of the `agent-skills` repo, locked in during brainstorming.)
- Not a debugger. If bugs surface mid-implementation, `gzship` references mattpocock's `/diagnose` six-phase loop but does not embed it.
- Not a CI/release tool. Once the final slice is green and reviewed, `gzship` terminates; pushing the PR / merging are out of scope (`babysit-pr` covers that).
- Not a code-review-of-existing-PRs tool. Use `critique-loop`'s review-only flow for that.
- Not an issue-tracker integration. PRD lives only as a file under `docs/features/<slug>/prd.md`. No GitHub issue is created.

## Phase overview

| # | Phase | Output | Gate |
|---|---|---|---|
| 1 | **Discover** | `docs/features/<slug>/scenarios.md` + (lazy) `CONTEXT.md` updates | — |
| 2 | **PRD** | `docs/features/<slug>/prd.md` | — |
| 3 | **Review A** | `critique-loop` review notes (gitignored) | `VERDICT: APPROVE` + user "go" |
| 4 | **Design** | `docs/features/<slug>/design.md` + `diagrams/current.{d2,svg}` + `diagrams/proposed.{d2,svg}` + new `docs/adr/NNNN-*.md` (if any) | — |
| 5 | **Review B** | `critique-loop` review notes (gitignored) | `VERDICT: APPROVE` + user "go" |
| 6 | **Slice** | `docs/features/<slug>/slices.md` | user "go" |
| 7 | **Implement** | code + tests per slice; per-slice `critique-loop` review | all slices green, all reviews `APPROVE` |

The slug is the current git branch name; if on `main`/`master`, the skill asks for one.

## Consumer-repo artifact layout

The first time `gzship` runs against a repo, it creates whatever is missing:

```
<repo-root>/
├── CONTEXT.md                        # lazily seeded on first resolved term
├── docs/
│   ├── adr/                          # lazily created when the first ADR is needed
│   │   └── NNNN-<slug>.md
│   └── features/<slug>/              # one directory per feature
│       ├── scenarios.md              # Phase 1
│       ├── prd.md                    # Phase 2
│       ├── design.md                 # Phase 4
│       ├── diagrams/
│       │   ├── current.d2            # Phase 4 source
│       │   ├── current.svg           # Phase 4 rendered (or .placeholder.md if d2 missing)
│       │   ├── proposed.d2
│       │   └── proposed.svg
│       └── slices.md                 # Phase 6
└── .critique-loop/                   # gitignored; critique-loop's working state
    └── <slug>-*.md                   # plan-review, code-review, session ids, etc.
```

`docs/features/<slug>/` is the unit of work. `docs/adr/` and `CONTEXT.md` are shared across features.

## Skill file structure (this repo)

```
skills/gzship/
├── SKILL.md                  # orchestrator: 7 phases, gates, subagent dispatch, escalation rules
└── references/
    ├── scenarios.md          # CONTEXT.md format + BDD scenarios (Gherkin, declarative, Three Amigos, Example Mapping, double-loop framing)
    ├── prd.md                # PRD template (Problem · Solution · User Stories · Implementation Decisions · Testing Decisions · Out of Scope) + how scenarios back stories
    ├── architecture.md       # Deep-modules vocab (module/interface/seam/adapter/depth/leverage/locality) + survey technique + deletion test + ADR offer criteria
    ├── implementation.md     # Vertical-slice tracer bullets (AFK vs HITL) + per-slice RED→GREEN + horizontal-slicing anti-pattern + subagent escalation (5+ slices)
    └── d2.md                 # D2 syntax for architecture diagrams (containers, sql_tables, scenarios for before/after) + install/fallback + rendering pipeline
```

SKILL.md is the orchestrator. Reference files are loaded only when the corresponding phase runs (progressive disclosure). The skill should fit comfortably in context when only SKILL.md is loaded.

## Phase detail

### Phase 1 — Discover

**Goal:** understand how this feature lands in the current product; write BDD scenarios that describe the desired behaviour.

**Steps:**

1. **Survey existing code.** Dispatch an `Explore` subagent to map the area of the codebase the feature will land in. Brief: "find the modules, callers, tests, and seams relevant to <feature>. Return a one-screen map; do not propose changes." Keeps main context lean.
2. **Read the glossary.** Open `CONTEXT.md` (if it exists). If a term in the user's prompt is missing or fuzzy, **grill the user one question at a time** (mattpocock's `grill-me` style) and **update `CONTEXT.md` inline** as terms resolve. Seed `CONTEXT.md` lazily on the first resolved term using the format in `references/scenarios.md`.
3. **Discover scenarios via Example Mapping** (Matt Wynne / Cucumber Book). For each user story, surface 1–3 concrete examples; turn each example into a Given/When/Then scenario.
4. **Write scenarios** to `docs/features/<slug>/scenarios.md` in Gherkin form. Rules (full detail in `references/scenarios.md`):
   - **Declarative**, not imperative — describe *what* the user achieves, not *which buttons* are clicked.
   - **One behaviour per scenario.** Scenarios that combine concerns get split.
   - **Use `CONTEXT.md` vocabulary** for every noun.
   - **Scenario outlines** for parameterised cases.
   - Mark each scenario with the user story it supports (`# Story: 3`).
5. **Commit:** `docs: scenarios for <slug>`.

**Subagent use:** parallel `Explore` subagents only — read-only surveys. No write access.

### Phase 2 — PRD

**Goal:** synthesise the conversation into a PRD without re-interviewing.

**Template** (per mattpocock's `to-prd`, full detail in `references/prd.md`):

- **Problem Statement** — from the user's perspective.
- **Solution** — from the user's perspective.
- **User Stories** — long numbered list, `As an <actor>, I want <feature>, so that <benefit>`. Each story lists which scenarios in `scenarios.md` verify it (e.g. `(verified by Scenarios 2, 4, 7)`).
- **Implementation Decisions** — modules to build/modify, interface shapes, schema changes, API contracts. No file paths or code snippets (exception: a decision-encoding snippet from a prototype, trimmed to decision-rich parts).
- **Testing Decisions** — what makes a good test for this feature; which modules will be tested; prior-art tests in the repo to follow.
- **Out of Scope** — explicit nos.

Commit: `docs: prd for <slug>`.

### Phase 3 — Review A (scenarios + PRD)

**Goal:** adversarial review of the product framing before any architecture work.

Run `critique-loop`'s review-only flow over the two committed files. The navigator is briefed to probe for:

- Scenarios that combine concerns and should be split.
- Scenarios that drift into implementation language.
- User stories with no scenario backing.
- Missing edge cases or error paths.
- Scope creep / scope ambiguity.

**Gate:** `VERDICT: APPROVE` from the navigator, then the user explicitly approves with `go`. If the user requests substantive changes, revise both files and re-run the navigator round (per `critique-loop`'s minor-vs-substantive split).

### Phase 4 — Design

**Goal:** show how the feature lands in the existing architecture and what changes.

**Steps:**

1. **Survey existing architecture.** Dispatch a second `Explore` subagent with a different brief: "map the architecture of <area> using `module / interface / seam / adapter` vocabulary. For each module, note its depth (leverage at the interface). Apply the deletion test to anything shallow."
2. **Draft `current.d2`** — the relevant slice of the existing architecture as it stands today. Use D2 containers for module groupings, arrows for dependencies. Render to `current.svg`.
3. **Draft `proposed.d2`** — the architecture after the feature lands. Prefer **D2 scenarios** (`scenario` syntax) so `current` and `proposed` can share a base file with the diff highlighted — but separate files are acceptable when scenarios get unwieldy. Render to `proposed.svg`.
4. **Visually verify** rendered SVGs (open and look at them; do not trust that D2 source rendered correctly).
5. **Write `design.md`:**
   - **Context** — one paragraph; what exists today.
   - **Proposal** — what changes; new/modified modules with their interfaces (just the interface — types, invariants, error modes, ordering — not the implementation).
   - **Deepening opportunities** — any shallow modules in the path that should be deepened as part of this work. Use mattpocock's vocabulary explicitly.
   - **Risks & trade-offs** — what got harder, what we're betting on.
   - **References to diagrams** — embed both `current.svg` and `proposed.svg`.
6. **Offer ADRs** for any decision that meets all three criteria: hard-to-reverse + surprising-without-context + result-of-real-trade-off. Use the tight ADR template (1–3 sentences body, optional sections only when they earn their place). Number sequentially from existing `docs/adr/`. Lazily create `docs/adr/` if missing.
7. **Commit:** `docs: design for <slug>` (one commit; include diagrams and any new ADRs).

**D2 install handling:** check `d2 --version`. If missing, offer to install (`brew install d2` on macOS, `curl -fsSL https://d2lang.com/install.sh | sh` elsewhere). If the user declines or install fails, write `current.d2` / `proposed.d2` source files only and commit a `<name>.placeholder.md` next to each with the rendering command — do not block the phase.

**Subagent use:** parallel `Explore` subagent for the architecture survey. Subagents may also be used to draft *alternative* interface designs for a deepened module (mattpocock's "Design It Twice" pattern — three parallel agents each given a different constraint: minimise / maximise flexibility / optimise common caller). Optional, used only when the design has a genuine "which interface?" question.

### Phase 5 — Review B (design)

Same shape as Review A. Navigator briefed to probe for:

- Proposed interfaces that are still shallow.
- Seams that exist for only one adapter (hypothetical seams).
- Implementation Decisions in the PRD that the design contradicts.
- Diagrams that disagree with the prose (or with the actual codebase).
- ADRs that should exist but don't (or do exist but shouldn't).

**Gate:** `VERDICT: APPROVE` + user "go".

### Phase 6 — Slice

**Goal:** decompose the design into vertical-slice tracer bullets.

**Rules** (full detail in `references/implementation.md`):

- Each slice is a **vertical** path through every layer (schema, API, UI, tests) — not a horizontal slice of one layer.
- Each slice is **independently demoable** when complete.
- Prefer **many thin slices** over few thick ones.
- Each slice is tagged **AFK** (no human gate) or **HITL** (requires user judgment mid-slice). Prefer AFK.
- Each slice lists what it depends on (slice IDs).

**Output:** `docs/features/<slug>/slices.md`. Each slice has:

- **Title**
- **Type** (AFK / HITL)
- **Blocked by** (slice IDs)
- **Stories covered** (PRD story numbers)
- **Acceptance criteria** (checklist — the contract for "done")
- **What to build** (concise behavioural description, no file paths)

**Gate:** user reviews and approves the slice list (`go`). The skill creates one TaskCreate entry per slice for the harness goal tracker.

### Phase 7 — Implement

**Goal:** ship the slices.

**Per-slice loop** (strict order, no horizontal slicing):

1. **RED:** write one failing test that asserts one behaviour from this slice's acceptance criteria. Test through the public interface only; integration-style; uses `CONTEXT.md` vocabulary in its name.
2. **GREEN:** minimal code to pass that one test.
3. **Repeat** RED→GREEN until every acceptance criterion has at least one test.
4. **Refactor on green only.** Apply deepening if the new code reveals a shallow module in the path. Run the suite after each refactor step.
5. **Commit** with conventional-commits messages; one logical change per commit.
6. **Run linters + tests** before declaring slice done.
7. **`critique-loop` review** of the slice diff (review-only flow, range = slice's commits). Address asks (code-level → Claude; product/architecture → user). Loop until `VERDICT: APPROVE`.
8. **Mark slice complete** in the task list. Move to the next slice.

**Horizontal slicing is explicitly forbidden.** Writing all tests first and then all the code produces tests that verify imagined behaviour and break under real refactors. The skill calls this out and references mattpocock's `tdd/SKILL.md` rationale.

**Subagent escalation (Approach 2):** when the feature has **5+ slices**, dispatch each slice to a fresh subagent. Brief contains: slice ID, acceptance criteria, references to `prd.md` and `design.md`, the `CONTEXT.md` path, and the deep-modules vocabulary file. Subagent does steps 1–6 and returns a summary; main session runs step 7 (`critique-loop`) and step 8 (task update). Sequential, not parallel — slices typically depend on each other and parallel subagents collide.

For fewer than 5 slices, the main session does everything.

## Orchestration model

**Linear orchestrator with two subagent uses:**

1. **Fan-out for read-only surveys** (Phases 1 and 4): parallel `Explore` subagents to read the code without bloating main context.
2. **Per-slice implementation** (Phase 7, escalated): one subagent per slice when a feature has 5+ slices.

**Not used:** maximal parallelism (Approach 3 from brainstorming) — single-feature stages are usually linearly dependent and parallel agents collide on shared files. The orchestrator stays linear.

**Task tracker as goal tracker:** SKILL.md tells the orchestrator to use `TaskCreate` to record one task per phase at start of run, and one task per slice during Phase 6. `TaskUpdate` marks each as `in_progress` / `completed`. This is the user's "goal tracker" view.

## Integration with existing tools

### `critique-loop` (this repo)

Every review gate (Phases 3, 5, and once per slice in Phase 7) invokes `critique-loop`'s **review-only** flow:

- Phase 3 review range: the `docs:` commits that added `scenarios.md` + `prd.md`.
- Phase 5 review range: the `docs:` commit that added `design.md` + diagrams + ADRs.
- Phase 7 per-slice review range: the slice's implementation commits.

Each review uses the same `<slug>` so all rounds share one navigator session (per `critique-loop`'s session-id reuse). The navigator therefore remembers the PRD and design context when it reviews each slice's diff — without re-reading.

### `CONTEXT.md` and `docs/adr/` (mattpocock conventions)

- `CONTEXT.md`: read in Phase 1 (Discover) and Phase 4 (Design). Updated inline as terms resolve. Lazily seeded on first resolved term (do not scaffold upfront). Format and rules in `references/scenarios.md`.
- `docs/adr/`: offered in Phase 4 (Design) when a decision meets all three criteria. Lazily created when the first ADR is needed. Sequential numbering. Format in `references/architecture.md`.

`gzship` does not depend on `mattpocock/skills` being installed. The relevant rules are inlined / referenced. Where the user has mattpocock's `/setup-matt-pocock-skills` already run, `gzship` happily uses what's there.

### D2 (diagrams)

- D2 chosen over Mermaid / PlantUML / ASCII during brainstorming.
- Architecture diagrams use D2 containers for module groupings, arrows for dependencies, and (when possible) D2 **scenarios** to express current → proposed as one base file with the diff highlighted.
- Pipeline: write `.d2` source → render to `.svg` via `d2 <in> <out>` → visually verify the SVG → embed in `design.md`.
- Fallback when `d2` binary missing: commit `.d2` source + a `.placeholder.md` with the render command. Do not block the phase.
- Full guidance in `references/d2.md`.

## Principles inherited from mattpocock/skills

These are stated in SKILL.md as load-bearing rules:

- **Ubiquitous language first.** Every doc gzship writes uses `CONTEXT.md` vocabulary. Drift = call out + propose canonical term.
- **ADRs are sparingly offered** — never as a checkbox; only when hard-to-reverse + surprising + real trade-off.
- **Vertical slices only.** Horizontal slicing (all tests, then all code) is explicitly forbidden.
- **Deletion test.** When surveying existing architecture, ask "if I deleted this, would complexity vanish or concentrate?"
- **The interface is the test surface.** Tests cross the same seam callers do. If you want to test past the interface, the module is the wrong shape.
- **One adapter = hypothetical seam. Two adapters = real seam.** Don't introduce ports for one consumer.
- **PRD contains no file paths or code snippets.** (Exception: prototype-derived decision-encoding snippet, trimmed to decision-rich parts.)
- **Deep modules**: small interface, lots of implementation. The design phase actively looks for shallow modules in the path and deepens them.

## Out of scope

- A portable variant (Cursor / Codex). Not part of v1. May be reconsidered later if there's demand.
- Multi-feature orchestration (e.g. "run gzship on this batch of features"). One feature at a time.
- Issue-tracker integration (GitHub/Linear publishing). PRD is a file. Maintainers who want issues can copy from the file.
- Live deployment / release / merge automation. `babysit-pr` covers post-merge concerns.
- A bug-fix flow. `gzship` is for greenfield features. For bugs use `diagnose` (mattpocock).

## Open questions

(None blocking. These can be deferred to implementation or v2.)

1. **D2 binary install:** is the install-prompt friction acceptable, or should the skill ship a hermetic D2 binary path? — defer.
2. **Subagent escalation threshold:** 5+ slices is a guess. Tune after first real runs.
3. **`critique-loop` session sharing across all 3 gates:** confirmed possible by `critique-loop`'s session-id reuse, but the prompt templates per gate need to be designed so the navigator doesn't get confused by the topic switch (scenarios → design → slice code).
4. **CONTEXT.md drift across features:** if two features add conflicting terms, the skill calls it out — but the resolution is the user's. No auto-merge.

## Next step

Hand off to the `writing-plans` skill to produce a step-by-step implementation plan for building `gzship` itself. The implementation plan should follow the same TDD-for-skills discipline that `writing-skills` requires: RED (baseline pressure scenarios on a subagent without the skill) → GREEN (write SKILL.md + reference files) → REFACTOR (close rationalisation loopholes).
