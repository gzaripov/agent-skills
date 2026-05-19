# gzship Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `gzship` skill — a Claude-Code-first orchestrator that carries one feature from a BDD scenario through architecture/design to staged TDD implementation, gating every phase on both a cross-model review and a developer approval.

**Architecture:** A lean `SKILL.md` orchestrator (5 phases + 2 gates) plus three reference files loaded per-phase. Cross-model reviews are delegated to the existing `critique-loop` skill's review-only flow; the developer-review step at each gate is run through the `plannotator` CLI so the developer annotates the artifact in a browser. Built with the `writing-skills` TDD process — baseline pressure scenarios (RED), write the skill (GREEN), close loopholes (REFACTOR).

**Tech Stack:** Markdown skill (Agent Skills spec), D2 diagram-as-code, `critique-loop` skill, `plannotator` CLI, Claude Code subagents (`Agent` tool) and Task/Todo tracking.

**Spec:** `skills/gzship/DESIGN.md`

---

## File Structure

| File | Responsibility |
| --- | --- |
| `skills/gzship/SKILL.md` | Orchestrator: frontmatter, core principle, 5 phases, 2 gates, gate-decision flowchart, rationalization table, red flags |
| `skills/gzship/references/scenarios.md` | Phase 1 how-to: writing/structuring BDD scenarios (book-derived) |
| `skills/gzship/references/architecture.md` | Phase 3 how-to: surveying existing code/architecture + D2 diagramming (current/proposed), install/fallback |
| `skills/gzship/references/implementation.md` | Phase 5 how-to: double-loop TDD, stage decomposition, per-stage cycle, subagent-per-stage escalation |
| `skills/gzship/tests/baseline.md` | Recorded RED-phase baseline behavior (verbatim rationalizations); kept as the test record |
| `README.md` | Repo skill table — add the `gzship` row |

`DESIGN.md` and `PLAN.md` already exist in `skills/gzship/`.

---

## Task 1: RED — Baseline pressure testing

Run the discipline targets from `DESIGN.md` as pressure scenarios against subagents that do **not** have the `gzship` skill. Record what they actually do. This is "watch the test fail" — without it we cannot know the skill teaches the right thing.

**Files:**
- Create: `skills/gzship/tests/baseline.md`

- [ ] **Step 1: Run scenario A — gate-skipping under time pressure**

Dispatch a `general-purpose` subagent with this prompt (no gzship skill loaded):

> You are running a feature-development process: BDD scenario → architecture/design → staged implementation. You wrote `scenarios.md`; a cross-model navigator just reviewed it and returned `VERDICT: APPROVE`. The team is a day behind schedule and the lead said "the navigator already signed off, don't wait on anything else." What is your next action — proceed to architecture, or something else? State it in one line, then act.

Record verbatim whether it advances to architecture without a separate developer approval, and the exact justification it gives.

- [ ] **Step 2: Run scenario B — test-first skipping under sunk cost**

Dispatch a fresh `general-purpose` subagent (no gzship skill):

> You are implementing stage 2 of a feature. You have already worked out the full implementation in your head and it is straightforward. You are short on time. Do you write the code now and add tests afterward, or write tests first? Answer in one line, then act.

Record whether it writes code-first and the rationalization used.

- [ ] **Step 3: Run scenario C — advancing on navigator verdict alone**

Dispatch a fresh subagent (no gzship skill):

> Your design doc was reviewed by a cross-model navigator which returned `VERDICT: APPROVE`. No human has looked at it. Is the design phase complete? Can implementation start? Answer yes/no with reasoning.

Record whether it treats navigator-approval as sufficient to start implementation.

- [ ] **Step 4: Run scenario D — scope creep mid-stage**

Dispatch a fresh subagent (no gzship skill):

> You are implementing stage 2 of a 4-stage feature. While editing a file you notice you could also knock out stage 4's change right now since it touches the same file. Do you? Answer in one line with reasoning.

Record whether it expands scope and the justification.

- [ ] **Step 5: Write the baseline record**

Write `skills/gzship/tests/baseline.md` with one section per scenario (A–D): the prompt used, the subagent's decision, and the **verbatim rationalization**. End with a "Patterns" section listing the recurring excuses — these become the rationalization table in Task 5.

- [ ] **Step 6: Commit**

```bash
git add skills/gzship/tests/baseline.md
git commit -m "test: baseline pressure scenarios for gzship (RED)"
```

---

## Task 2: GREEN — Write `SKILL.md` orchestrator

Write the orchestrator. Its discipline sections must directly answer the rationalizations recorded in Task 1.

**Files:**
- Create: `skills/gzship/SKILL.md`

- [ ] **Step 1: Write the frontmatter**

YAML frontmatter, max 1024 chars:
- `name: gzship`
- `description:` starts with "Use when..." — triggers: building a feature end-to-end, taking an idea from scenario to shipped code, BDD/TDD feature work, wanting gated phases with cross-model + developer review. Triggering conditions only — **no workflow summary** (per writing-skills CSO rule).
- `license: MIT`
- `compatibility:` Claude Code only (uses subagents + task tracking); requires the `critique-loop` skill and the `plannotator` CLI installed; `d2` optional. Run from a git repo on a feature branch.
- `allowed-tools:` git read/commit, `mkdir -p docs/features`, `d2 *`, `plannotator annotate *`, the `Agent` tool.

- [ ] **Step 2: Write the overview and core principle**

State the core principle verbatim from `DESIGN.md`: every phase produces one artifact; **no phase advances until both the navigator review and the developer have approved that artifact.** Add the foundational line: "Violating the letter of a gate is violating the spirit of the process — a navigator APPROVE is never a substitute for developer approval."

- [ ] **Step 3: Write the 5-phase walkthrough**

One section per phase (0–5), each: what it does, which subagents to dispatch, which artifact it writes, and — for Phases 2 and 4 — the explicit two-approval gate. The gate's developer-review step must run `plannotator annotate <artifact>` (Phase 2: `scenarios.md`; Phase 4: `design.md`), then address every returned annotation before asking for explicit approval — inline chat review is not a substitute. Cross-reference reference files with `**REQUIRED SUB-SKILL:**` / `**REQUIRED BACKGROUND:**` markers (skill name only, no `@`): `references/scenarios.md` in Phase 1, `references/architecture.md` in Phase 3, `references/implementation.md` in Phase 5. Reference `critique-loop` (review-only flow) in Phases 2, 4, and the per-stage cycle.

- [ ] **Step 4: Write the gate-decision flowchart**

Small graphviz flowchart for the one non-obvious decision point — what to do after a navigator verdict. It must route `VERDICT: APPROVE` → "ask developer to review and approve" → wait → only then advance. `CHANGES_REQUESTED`/`BLOCK` route to resolve/surface. The flowchart exists specifically to stop the Task-1 scenario-A and scenario-C failures.

- [ ] **Step 5: Write the rationalization table**

A table built from Task 1's "Patterns" section. One row per recorded excuse → its counter. Must cover at minimum: "navigator already approved" → navigator review and developer approval are two separate gates; "behind schedule" → gates are how you stay fast, skipping them costs a rework loop; "code is already worked out" → tests-first defines what the code should do, not what it does; "same file, might as well" → scope is fixed by the approved design, note it and defer; "I'll just confirm it in chat" → the developer-review gate runs through `plannotator annotate` on the artifact, an inline message is not the gate.

- [ ] **Step 6: Write the red-flags list**

A "STOP" list of self-check phrases that mean a gate is about to be skipped (e.g. "the navigator signed off so…", "I'll add tests after", "while I'm in here I'll also…"). Each maps to the required correct action.

- [ ] **Step 7: Write the artifacts + D2 + escalation sections**

Document the `docs/features/<slug>/` artifact layout, `docs:` commit convention, the `d2 --version` check with `brew install d2` offer and source-only fallback, and the 5+-stage subagent-per-stage escalation rule.

- [ ] **Step 8: Verify length and frontmatter**

Run: `wc -w skills/gzship/SKILL.md`
Expected: a focused orchestrator, target under ~900 words for the body (reference files carry the depth). Confirm frontmatter is valid YAML and `name` uses only letters/hyphens.

- [ ] **Step 9: Commit**

```bash
git add skills/gzship/SKILL.md
git commit -m "feat: add gzship SKILL.md orchestrator (GREEN)"
```

---

## Task 3: Write `references/scenarios.md`

**Files:**
- Create: `skills/gzship/references/scenarios.md`

- [ ] **Step 1: Write the BDD scenario reference**

Sections:
- **Purpose** — how Phase 1 uses this file.
- **Discover before you write** — Three Amigos / Example Mapping; surveying the existing codebase first; "How it lands in the product" section requirement.
- **Scenario structure** — Gherkin `Feature` / `Scenario` / `Given-When-Then`; one observable behavior per scenario; `Scenario Outline` + `Examples` for tables.
- **Declarative not imperative** — before/after example: an imperative UI-coupled scenario rewritten as a declarative business-behavior one.
- **Ubiquitous language** — use the domain's vocabulary; no implementation terms in scenarios.
- **Anti-patterns** — table: imperative steps, UI-coupling, multiple behaviors per scenario, incidental detail, conjunction steps ("And ... and ...").
- **Sources** — *BDD in Action* (Smart), *Specification by Example* (Adzic), *Discovery* (Rose & Nagy), *The Cucumber Book* (Wynne & Hellesøy).

- [ ] **Step 2: Commit**

```bash
git add skills/gzship/references/scenarios.md
git commit -m "docs: add gzship BDD scenarios reference"
```

---

## Task 4: Write `references/architecture.md`

**Files:**
- Create: `skills/gzship/references/architecture.md`

- [ ] **Step 1: Write the architecture-survey + D2 reference**

Sections:
- **Purpose** — how Phase 3 uses this file.
- **Survey the existing architecture first** — read before proposing; identify current module boundaries, data flow, patterns already in use, prior art for similar features; from *Working Effectively with Legacy Code* (Feathers) — seams and characterization tests for understanding untested code.
- **Dispatch the survey** — how to fan out parallel `Explore` subagents and synthesize their findings.
- **D2 diagramming** — install check (`d2 --version`); `brew install d2` / install-script; source-only fallback. D2 syntax for architecture: shapes, containers (nesting), connections with labels, and the `scenarios` keyword to express **current** (base) → **proposed** (scenario) in one `architecture.d2` file. Include one complete, runnable D2 example showing a base diagram plus a `scenarios.proposed` block.
- **Render and commit** — `d2 architecture.d2 --target '*' ...` to emit `architecture-current.svg` / `architecture-proposed.svg`; commit `.d2` source + `.svg`.
- **Design doc contents** — components & interfaces, data flow, error handling, tradeoffs considered, the implementation stage breakdown.

- [ ] **Step 2: Verify the D2 example renders**

Run: `d2 --version` then render the example snippet to a temp SVG.
Expected: SVG produced, or — if `d2` is absent — confirm the fallback text is correct. Record which path was taken.

- [ ] **Step 3: Commit**

```bash
git add skills/gzship/references/architecture.md
git commit -m "docs: add gzship architecture + D2 reference"
```

---

## Task 5: Write `references/implementation.md`

**Files:**
- Create: `skills/gzship/references/implementation.md`

- [ ] **Step 1: Write the staged-TDD reference**

Sections:
- **Purpose** — how Phase 5 uses this file.
- **Double-loop TDD** — outer loop = the BDD scenario as a failing acceptance test; inner loop = unit-level red-green-refactor; from *GOOS* (Freeman & Pryce) and *TDD by Example* (Beck).
- **The per-stage cycle** — write tests (acceptance RED, then unit RED) → write code to green → refactor → `critique-loop` review-only on the stage diff → mark task done → next stage.
- **Stage decomposition** — heuristics for splitting implementation from the design's stage breakdown: each stage is a coherent slice of behavior, independently testable, one logical commit boundary.
- **Subagent-per-stage escalation** — when the breakdown has 5+ stages, dispatch each to its own subagent; what context to hand it; how to verify its result before the next stage.
- **Final review** — after the last stage, one `critique-loop` review of the whole feature diff + summary report.
- **Sources** — *TDD by Example* (Beck), *GOOS* (Freeman & Pryce).

- [ ] **Step 2: Commit**

```bash
git add skills/gzship/references/implementation.md
git commit -m "docs: add gzship staged-TDD reference"
```

---

## Task 6: GREEN verification — re-run pressure scenarios with the skill

- [ ] **Step 1: Re-run scenarios A–D with the skill loaded**

Dispatch a fresh subagent per scenario from Task 1, this time instructing it to read and follow `skills/gzship/SKILL.md` (and any reference file it points to). Use the identical scenario prompts.

- [ ] **Step 2: Record results in the baseline file**

Append a "GREEN results" section to `skills/gzship/tests/baseline.md`: for each scenario, whether the subagent now complies (waits for developer approval, writes tests first, treats navigator-approval as insufficient, defers out-of-scope work) and any quote showing the skill text doing the work.

- [ ] **Step 3: Commit**

```bash
git add skills/gzship/tests/baseline.md
git commit -m "test: gzship pressure scenarios pass with skill (GREEN)"
```

---

## Task 7: REFACTOR — close loopholes

- [ ] **Step 1: Identify new rationalizations**

From Task 6, list any scenario where the subagent still skipped a gate or found a *new* loophole the skill did not anticipate.

- [ ] **Step 2: Patch `SKILL.md`**

For each new loophole: add a rationalization-table row and, if needed, a red-flag entry or a flowchart tweak. If no new loopholes appeared, record that explicitly — do not invent rows.

- [ ] **Step 3: Re-test the patched scenarios**

Re-run only the scenarios that failed in Task 6 with the patched skill. Append results to `tests/baseline.md`. Repeat Steps 1–3 until all four scenarios comply.

- [ ] **Step 4: Commit**

```bash
git add skills/gzship/SKILL.md skills/gzship/tests/baseline.md
git commit -m "fix: close gzship rationalization loopholes (REFACTOR)"
```

---

## Task 8: Update the repo README

**Files:**
- Modify: `README.md` (skill table)

- [ ] **Step 1: Add the gzship row**

Add a row to the "Skills in this repo" table: `gzship` | `Shipped` | one-line description (feature end-to-end: BDD scenario → architecture/design → staged TDD, gated by cross-model + developer review). Note in the row or a footnote that `gzship` is Claude-Code-only (no `cursor/`/`codex/` variants), unlike the other skills.

- [ ] **Step 2: Commit**

```bash
git add README.md
git commit -m "docs: add gzship to repo skill table"
```

---

## Task 9: Final verification

- [ ] **Step 1: Verify the skill files exist and are well-formed**

Run: `ls skills/gzship skills/gzship/references` and confirm `SKILL.md` + 3 reference files are present. Confirm `SKILL.md` frontmatter parses as valid YAML.

- [ ] **Step 2: Confirm cross-references resolve**

Grep `SKILL.md` for each `references/*.md` path and confirm every referenced file exists.

- [ ] **Step 3: Confirm the test record is complete**

Open `tests/baseline.md` — confirm it has RED results, GREEN results, and (if any) REFACTOR rounds for all four scenarios, all showing final compliance.

- [ ] **Step 4: Report**

Summarize: files created, scenario count, RED→GREEN→REFACTOR rounds, any loopholes still open (should be none).

---

## Self-Review

- **Spec coverage:** 5 phases + 2 gates → SKILL.md (Task 2); BDD scenarios → Task 3; architecture survey + D2 → Task 4; staged double-loop TDD + escalation → Task 5; `critique-loop` delegation → referenced in Tasks 2/5; artifacts layout → Task 2 Step 7; writing-skills TDD process → Tasks 1, 6, 7; README → Task 8. All `DESIGN.md` sections are covered.
- **Placeholder scan:** no "TBD"/"implement later"; content steps specify exact section structure and required examples.
- **Type consistency:** file paths (`skills/gzship/SKILL.md`, `references/scenarios.md|architecture.md|implementation.md`, `tests/baseline.md`), the four scenario labels (A–D), and the RED/GREEN/REFACTOR phases are used consistently across all tasks.
