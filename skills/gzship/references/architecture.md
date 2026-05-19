# Surveying architecture & writing the design — `gzship` Phase 3 reference

## Purpose

Phase 3 produces `docs/features/<slug>/design.md` plus an architecture diagram
showing both the **current** and **proposed** architecture. The design doc is the
input to the Phase 4 review gate: it must let a reviewer understand *why* this
design, not just *what* it is.

This file has two parts: **Rules** every design must obey, and **Recommendations**
applied with judgment. Follow the rules always.

## Rules — always

A design doc that breaks one of these fails the gate.

1. **Survey the existing architecture before proposing a change.** Describe the
   system as-is — module boundaries, data flow, resident patterns — before the
   to-be. Never design against a system you have not read.
2. **State Goals and Non-Goals explicitly.** Non-goals are reasonable objectives
   deliberately excluded; naming them forces a real scope decision.
3. **Every design element traces to an approved scenario.** Anything in the design
   that no scenario calls for is scope creep — cut it or justify it.
4. **State the alternatives considered — including "do nothing" — and why this one
   won.** A design with no alternatives section is not reviewable.
5. **Be tradeoff-centric.** For the chosen design, state what it *costs*, not only
   what it buys. A doc that only sells is defective.
6. **Make assumptions and failure modes explicit.** Write down dependencies, load,
   and data-shape assumptions; enumerate how the design can fail and how each
   failure is handled or knowingly accepted.
7. **Surface unresolved open questions.** List them plainly; never bury uncertainty
   in prose or silently design past it. Product/architecture questions go to the
   developer at the gate.

## Recommendations — apply with judgment

- **Size the doc to the change** — a one-page mini-doc for a small feature; long
  form only for large ones. As short as possible, as long as necessary.
- **Lead with a one-paragraph overview / TL;DR** before the detailed design.
- **Show current vs. proposed side by side** — paired diagrams make the delta
  reviewable at a glance.
- **Sketch APIs and data shapes, don't paste them** — include only the parts that
  drive a tradeoff; link full schemas.
- **For legacy or untested code, name the seams and a characterization-test plan** —
  how you will pin current behavior before changing it.
- **Record architecture-significant decisions as ADRs** — Context / Decision /
  Consequences; one per decision; immutable once accepted.
- **Address cross-cutting concerns** — security, observability, data migration —
  or mark each an explicit "N/A because…".

## Survey the existing architecture first

You cannot propose a sound change to a system you have not read. Before drawing a
box, pin down: **module boundaries** (what owns what, where the seams are),
**data flow** (how a request moves, what it carries), **resident patterns** (the
conventions the code already follows), and **prior art** (the closest existing
feature — copy its shape).

When the area is **untested or unfamiliar**, use *Working Effectively with Legacy
Code* (Michael Feathers):

- **Seams** — a place where you can change behavior without editing code there (a
  function boundary, an interface, an injection point). Existing seams show where a
  feature can attach without invasive surgery; a missing seam shows where risk
  concentrates.
- **Characterization tests** — when code has no tests and its behavior is not
  obvious, write tests that pin *what it currently does*. They turn an unfamiliar
  module into a known quantity before any change.

## Dispatch the survey

The survey fans out — do not read the whole codebase serially in the main context.

1. **Split the question** into independent sub-questions: module boundaries,
   data-flow path, resident patterns, closest prior art.
2. **Fan out parallel `Explore` subagents** — one per sub-question, in a single
   batch. Ask each for file paths plus a short synthesis, not raw dumps.
3. **Synthesize** the findings into one picture; reconcile conflicts by reading the
   specific files yourself.
4. **Feed the synthesis** into the diagram's *current* state and the opening
   context of `design.md`.

## Diagramming — draft both tools, render both, choose

`gzship` is not tied to one diagram tool. For the architecture diagram, **draft it
in both D2 and Mermaid, render each, look at the two renders, and keep whichever
reads more clearly for this feature.** Neither tool wins every time:

- **D2** — stronger auto-layout for dense graphs and nested containers; one file
  can hold both states (`layers` / `scenarios`). Needs the `d2` binary, and GitHub
  does not render `.d2` — the rendered SVG must be committed and embedded.
- **Mermaid** — GitHub and most markdown viewers render a fenced ` ```mermaid `
  block inline, so the diagram needs no committed image. Cleaner default shapes for
  simple flows; but it wraps long labels aggressively and has no current-vs-proposed
  board (use two diagrams).

Diagram sources live in `docs/features/<slug>/diagrams/`.

### D2

Install check: `d2 --version`. If missing, offer `brew install d2` or
`curl -fsSL https://d2lang.com/install.sh | sh -`.

- **Shape** — `api: API Gateway`, or `db: Postgres { shape: cylinder }`. Pick a
  fitting shape: a plain rectangle for a normal module, a cylinder only for a
  datastore.
- **Container** — dotted keys or nested braces group shapes: `backend.api`.
- **Connection** — `a -> b: label`; edges may cross containers.
- **Two states in one file** — `scenarios` blocks inherit the base and override
  (proposed builds on current); `layers` blocks are independent boards (current and
  proposed share no structure, e.g. a greenfield repo).

Render each board with `--target` — `''` for the base, `scenarios.<name>` or
`layers.<name>` for the proposed board:

```sh
D=docs/features/<slug>/diagrams
d2 --target ''                "$D/architecture.d2" "$D/architecture-current.svg"
d2 --target 'layers.proposed' "$D/architecture.d2" "$D/architecture-proposed.svg"
```

### Mermaid

Render check: `mmdc --version`, or run on demand with `npx -y
@mermaid-js/mermaid-cli`. A `flowchart` is the usual fit:

- **Node shapes** — `id["box"]`, `id(["pill / actor"])`, `id{{"hexagon"}}`,
  `id[("cylinder")]`.
- **Edge with label** — `a -- "label" --> b`.
- **Line breaks** — `<br/>` inside a label; keep each line short, Mermaid wraps
  long lines at awkward points.
- **Two states** — current and proposed are two separate ` ```mermaid ` blocks.

Render to PNG to inspect: `npx -y @mermaid-js/mermaid-cli -i "$D/architecture.mmd"
-o /tmp/arch.png -s 2`.

### Look at what you rendered — and compare

A diagram you have not seen is not done. Render **both** versions to PNG and open
them (you can read image files). Judge each as a reader would:

- no oversized or near-empty shapes; no node far larger than its content;
- no overlapping nodes or labels; no clipped or cramped text;
- few edge crossings; related nodes sit close together;
- the layout is balanced — not jammed into one corner with dead space.

Fix the source and re-render until each reads cleanly. Then **pick the tool whose
render is clearer** — that one becomes the diagram.

## Embed and commit

Embed the **chosen** diagram in `design.md` so the doc is self-contained:

- **Mermaid chosen** — paste it into `design.md` as a fenced ` ```mermaid ` block
  (current and proposed each their own block). GitHub renders it inline; no image
  artifact needed. Keep the source as `diagrams/architecture.mmd` too.
- **D2 chosen** — embed the rendered SVGs with markdown image syntax (never a raw
  `.d2`); commit the `.d2` source and both `.svg` files.

Commit with a `docs:` conventional commit. If neither tool installs, commit the
diagram source and embed it as a fenced code block so it is reviewable as text.

## The design document

`docs/features/<slug>/design.md` must contain, anchored to the survey:

- **Context & scope** — the problem and the current architecture it lands in.
- **Goals / Non-Goals** — explicit bullets (Rule 2).
- **Components & interfaces** — the units added or changed, with the contract
  (signature, inputs, outputs) at each boundary.
- **Data flow** — the path a request/event takes through the proposed design.
- **Diagrams** — current and proposed, embedded (see above).
- **Alternatives considered** — incl. "do nothing", with rejection reasons (Rule 4).
- **Tradeoffs** — what the chosen design costs (Rule 5).
- **Error handling & failure modes** — how each failure is detected and handled.
- **Assumptions & open questions** — explicit (Rules 6, 7).
- **Implementation stage breakdown** — the feature split into ordered,
  independently testable stages, each small enough for one double-loop TDD cycle.
  Drives Phase 5; if it has **5 or more stages**, each is dispatched to its own
  subagent.

## Anti-patterns

| Smell | Why it hurts | Fix |
|---|---|---|
| Implementation manual — *how*, no *why* or tradeoffs | Not reviewable; reviewer can't judge the choice | Add design rationale and tradeoffs; if there's no ambiguity, skip the doc |
| No alternatives considered | The choice looks inevitable; reviewer can't push back | Add ≥2 alternatives incl. "do nothing" and rejection reasons |
| Vague requirements ("fast", "scalable") | Unverifiable; means nothing at the gate | Replace adjectives with measurable targets tied to a scenario |
| Unstated assumptions | Hidden risk; the design breaks when one is wrong | List every assumption; mark validated vs. unverified |
| Scope creep | The doc balloons into adjacent systems | Add Non-Goals; move tangential concerns out |
| No failure analysis | Only the happy path is designed | Enumerate failure modes and the handling for each |
| Designs against an unknown system | Proposes change without surveying current state | Document the as-is architecture first; for untested code, name seams |
| Diagram with no legend/labels | Boxes and arrows that need narration | Title, key, labeled directional edges; verify the render |
| Buried uncertainty | Open questions hidden in prose or omitted | Hoist them into an Open Questions section |
| Untraceable design | Elements with no originating scenario | Drop or justify any element that doesn't trace to an approved scenario |

## Sources

- *Working Effectively with Legacy Code* — Michael Feathers: seams, characterization
  tests.
- "Design Docs at Google" — Malte Ubl: context/scope, goals/non-goals, alternatives,
  tradeoff-centric design, cross-cutting concerns.
- Architecture Decision Records — Michael Nygard; Martin Fowler: Context / Decision
  / Consequences, immutable once accepted.
- The C4 model — Simon Brown: context diagrams, self-describing notation.
- D2 ([d2lang.com](https://d2lang.com)) and Mermaid ([mermaid.js.org](https://mermaid.js.org)):
  diagram languages.
