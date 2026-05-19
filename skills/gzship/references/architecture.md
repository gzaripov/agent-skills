# Surveying architecture & diagramming — `gzship` Phase 3 reference

## Purpose

Phase 3 produces `docs/features/<slug>/design.md` plus an architecture diagram (in
`diagrams/`) showing both the **current** and **proposed** architecture. This file is
the standard that work is held to. Use it while dispatching the architecture survey,
drawing the diagrams, and writing the design doc — survey the system as it *is* before
proposing how it *should* change, and show both states.

## Survey the existing architecture first

You cannot propose a sound change to a system you have not read. Before drawing a
single box, survey the codebase as it exists and pin down:

- **Module boundaries** — what units own what responsibility, and where the seams
  between them sit.
- **Data flow** — how a request/event moves through those modules, what it carries,
  and where it is transformed or persisted.
- **Design patterns already in use** — the conventions the code already follows
  (layering, dependency injection, event bus, repository, etc.). A proposal that
  ignores the resident patterns gets rejected at the design gate.
- **Prior art for similar features** — find the closest existing feature and copy its
  shape. Consistency with what exists beats a locally clever design.

When the area you must change is **untested or unfamiliar**, use the techniques from
*Working Effectively with Legacy Code* (Michael Feathers) to understand it before
proposing changes:

- **Seams.** A *seam* is a place where you can alter behavior without editing the code
  at that spot — a function boundary, an interface, an injection point. Identifying the
  existing seams tells you where a new feature can attach without invasive surgery, and
  where you would have to *create* a seam (and therefore where risk concentrates).
- **Characterization tests.** When code has no tests and its behavior is not obvious,
  write tests that *characterize* what it currently does — pin the observed behavior,
  not the intended behavior. These tests turn an unfamiliar module into a known
  quantity and become a safety net before any change. In `gzship` they feed Phase 3
  understanding; the feature's own behavior tests are written later, in Phase 5.

Record the survey result in the design doc so the proposed change is visibly anchored
to the system that exists.

## Dispatch the survey

The survey fans out — do not read the whole codebase serially in the main context.

1. **Split the question.** Turn the survey into independent sub-questions, e.g.
   "module boundaries and ownership in area X", "the data-flow path for the relevant
   request", "design patterns and conventions used here", "the closest prior-art
   feature and how it is structured".
2. **Fan out parallel `Explore` subagents** — one per sub-question, dispatched in a
   single batch so they run concurrently. Give each a tight prompt and ask for file
   paths plus a short synthesis, not raw dumps.
3. **Synthesize.** Merge the findings into one coherent picture: current boundaries,
   current data flow, resident patterns, prior art. Reconcile conflicts between
   subagent reports by reading the specific files yourself.
4. **Feed the synthesis into both deliverables** — it becomes the *current* state of
   `architecture.d2` and the opening context of `design.md`.

## Diagramming — draft both tools, render both, choose

`gzship` is not tied to one diagram tool. For the architecture diagram, **draft it in
both D2 and Mermaid, render each, look at the two renders, and keep whichever reads
more clearly for this feature.** Neither tool wins every time:

- **D2** — stronger auto-layout for dense graphs and nested containers; one file can
  hold both the current and proposed states (`layers` / `scenarios`). Needs the `d2`
  binary, and GitHub does not render `.d2` — the rendered SVG must be committed and
  embedded.
- **Mermaid** — GitHub and most markdown viewers render a fenced ` ```mermaid ` block
  inline, so the diagram needs no committed image and stays in `design.md` as text.
  Cleaner default shapes for simple flows; but it wraps long labels aggressively and
  has no current-vs-proposed board (use two diagrams).

Diagram sources live in `docs/features/<slug>/diagrams/`.

### D2

Install check: `d2 --version`. If missing, offer `brew install d2` or
`curl -fsSL https://d2lang.com/install.sh | sh -`.

Syntax essentials:

- **Shape** — `api: API Gateway`, or `db: Postgres { shape: cylinder }`. Pick a
  fitting shape: a plain rectangle for a normal module, a cylinder only for a
  datastore.
- **Container** — dotted keys or nested braces group shapes: `backend.api` and
  `backend.worker` sit inside a drawn `backend` box.
- **Connection** — `a -> b: label`; `--` undirected, `<->` bidirectional; edges may
  cross containers.
- **Two states in one file** — `scenarios` blocks each inherit the base and add or
  override on top (use when proposed builds on current); `layers` blocks are
  independent boards (use when current and proposed share no structure, e.g. a
  greenfield repo).

Render each board to its own SVG with `--target` — `''` for the base/current board,
`scenarios.<name>` or `layers.<name>` for the proposed board:

```sh
D=docs/features/<slug>/diagrams
d2 --target ''                "$D/architecture.d2" "$D/architecture-current.svg"
d2 --target 'layers.proposed' "$D/architecture.d2" "$D/architecture-proposed.svg"
```

### Mermaid

Render check: `mmdc --version`, or run it on demand with `npx -y
@mermaid-js/mermaid-cli`. A `flowchart` is the usual fit:

- **Node shapes** — `id["box"]`, `id(["pill / actor"])`, `id{{"hexagon"}}`,
  `id[("cylinder")]`.
- **Edge with label** — `a -- "label" --> b`.
- **Line breaks** — use `<br/>` inside a label and keep each line short; Mermaid
  auto-wraps long lines at awkward points.
- **Two states** — write the current and proposed flows as two separate
  ` ```mermaid ` blocks; Mermaid has no board concept.

Render to PNG to inspect it:

```sh
npx -y @mermaid-js/mermaid-cli -i "$D/architecture.mmd" -o /tmp/arch.png -s 2
```

### Look at what you rendered — and compare

A diagram you have not seen is not done. Render **both** the D2 and the Mermaid
version to PNG and open them (you can read image files). Judge each as a reader would:

- no oversized or near-empty shapes; no node far larger than its content;
- no overlapping nodes or labels; no clipped or cramped text;
- few edge crossings; related nodes sit close together;
- the layout is balanced — not jammed into one corner with dead space elsewhere.

Fix the source and re-render until each reads cleanly. Then **pick the tool whose
render is clearer for this feature** — that one becomes the diagram.

## Embed and commit

Embed the **chosen** diagram in `design.md` so the design doc is self-contained:

- **Mermaid chosen** — paste the diagram into `design.md` as a fenced ` ```mermaid `
  block (current and proposed each as their own block). GitHub renders it inline; no
  image artifact is needed. Keep the source as `diagrams/architecture.mmd` as well.
- **D2 chosen** — embed the rendered SVGs with markdown image syntax (never a raw
  `.d2`), and commit the `.d2` source plus both `.svg` files:

  ```markdown
  ![Current architecture](./diagrams/architecture-current.svg)
  ![Proposed architecture](./diagrams/architecture-proposed.svg)
  ```

Commit the diagram with a `docs:` conventional commit. If neither tool can be
installed, fall back to committing the diagram source and embedding it as a fenced
code block so it is at least reviewable as text.

## The design document

`docs/features/<slug>/design.md` is the Phase 3 artifact and the input to the Phase 4
review gate. It must contain:

- **Components & interfaces** — the units the feature adds or changes, and the contract
  (signature, inputs, outputs) at each boundary. Anchor each to the surveyed current
  architecture and the resident patterns.
- **Data flow** — how data moves through the proposed design: the path a request/event
  takes, what it carries, where it is transformed or persisted. Reference the embedded
  current and proposed diagrams.
- **Error handling** — failure modes, how each is detected, and the recovery or
  fallback behavior. Cover the seams the change introduces.
- **Tradeoffs considered** — the alternatives weighed and why this design was chosen;
  cost, risk, and consistency with prior art. Surface unresolved architecture/product
  tradeoffs as open questions for the developer gate.
- **Implementation stage breakdown** — the feature split into ordered, independently
  testable stages. Each stage names the behavior it delivers and is small enough for
  one double-loop TDD cycle. This breakdown drives Phase 5; if it has **5 or more
  stages**, each stage is dispatched to its own fresh subagent.

Embed the chosen diagram — a ` ```mermaid ` block, or the rendered D2 SVGs — so the
design doc is self-contained for review.

## Sources

- *Working Effectively with Legacy Code* — Michael Feathers: seams, characterization
  tests, understanding existing/untested code before changing it.
- D2 — Terrastruct: declarative diagram language, `scenarios` / `layers` for
  multi-board diagrams ([d2lang.com](https://d2lang.com)).
- Mermaid: markdown-native diagram language rendered inline by GitHub and most
  viewers ([mermaid.js.org](https://mermaid.js.org)).
